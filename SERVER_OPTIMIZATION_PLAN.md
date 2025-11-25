# Groovy Language Server - Optimization & Improvement Plan

## Executive Summary

The LSP4Groovy server is functionally complete but has **2-5x performance optimization opportunity**. Response times can be reduced from 200-1000ms to 50-200ms by fixing thread safety issues, implementing intelligent caching, and optimizing AST operations.

**Current Performance Baseline** (estimated on large projects):
- Autocomplete: 300-800ms
- Hover: 150-500ms  
- File switch: Adds 100-500ms stall
- Initialization: 30-60s (blocked by Gradle import)

**Target After Optimization**:
- Autocomplete: 50-150ms
- Hover: 20-50ms
- File switch: No noticeable delay
- Initialization: 5-10s (with caching)

---

## 1. Architecture Overview

```
TCP/Stdio Server (GroovyLanguageServer.java)
    ↓
Groovy Services (GroovyServices.java)
    - Compilation Unit Factory
    - File Contents Tracker
    - AST Node Visitor
    ↓
Providers (9 implementations)
    - CompletionProvider (uses ClassGraph scan)
    - DefinitionProvider
    - HoverProvider
    - SignatureHelpProvider
    - ReferenceProvider
    - RenameProvider
    - TypeDefinitionProvider
    - DocumentSymbolProvider
    - WorkspaceSymbolProvider
    ↓
Classpath Resolution
    - Gradle Tooling API (discovers dependencies)
    - ClassGraph (scans classes at runtime)
```

**Key Components**:
- **Thread Synchronization**: All operations protected by `lock` object
- **File Tracking**: `FileContentsTracker` maintains open files
- **AST Caching**: `ASTNodeVisitor` builds node→URI→Range mappings
- **ClassPath Scanning**: `ClassGraph` scans runtime classes for autocomplete

---

## 2. Performance Bottlenecks (Detailed Analysis)

### 2.1 ClassGraph Full Scan - **CRITICAL** (500ms-2s per update)

**Problem**: Every classpath change triggers full classpath rescan
- Location: `GroovyServices.java:424-430`
- Called on: Initialization, classpath update, compilation unit change
- Scope: All system jars + project jars + modules

**Root Cause**:
```java
classGraphScanResult = new ClassGraph().overrideClassLoaders(classLoader)
    .enableClassInfo()
    .enableSystemJarsAndModules()  // ← Scans hundreds of JARs
    .scan();
```

**Impact**:
- 500ms-2000ms per scan
- Blocks server startup
- Called during file opens that update classpath

**Solution**: 
1. Cache scan result with classpath hash (avoid re-scanning identical classpath)
2. Make system jars optional (configurable, off by default)
3. Add scan timeout to prevent hangs
4. Implement async scan for background updates

---

### 2.2 AST Node Lookup - **HIGH** (50-200ms per request)

**Problem**: Linear O(n) search through all AST nodes with repeated Range creation
- Location: `ASTNodeVisitor.java:getNodeAtLineAndColumn()`
- Called on: Every completion, hover, definition request
- Complexity: O(n) where n = nodes in file (1000+ for large files)

**Root Cause**:
```java
List<ASTNode> foundNodes = nodes.stream()
    .filter(node -> {
        Range range = GroovyLanguageServerUtils.astNodeToRange(node);  // ← REPEATED
        return Ranges.contains(range, position);
    })
    .sorted((n1, n2) -> {
        Range r1 = GroovyLanguageServerUtils.astNodeToRange(n1);  // ← REPEATED AGAIN
        Range r2 = GroovyLanguageServerUtils.astNodeToRange(n2);  // ← REPEATED AGAIN
        // Complex multi-level sorting
    })
    .collect(Collectors.toList());
```

**Impact**:
- Creates thousands of Range objects per request
- Complex sorting overhead on every request
- No spatial indexing (tree/grid structure)

**Solution**:
1. Cache Range objects in AST node visit phase (use WeakHashMap for GC)
2. Replace linear search with spatial index (grid or quad-tree)
3. Pre-sort by line number during visit

---

### 2.3 Full Recompilation on File Context Switch - **HIGH** (100-500ms stall)

**Problem**: Switching between open files triggers full recompilation
- Location: `GroovyServices.java:recompileIfContextChanged()`
- Happens on: Every request from different file
- Cost: Full compilation to CANONICALIZATION phase

**Root Cause**:
```java
protected void recompileIfContextChanged(URI newContext) {
    if (previousContext == null || !previousContext.equals(newContext)) {
        fileContentsTracker.forceChanged(newContext);
        compileAndVisitAST(newContext);  // ← FULL RECOMPILE
    }
}
```

**Impact**:
- Noticeable 100-500ms delay when switching files
- Scales with project size
- Happens even for simple hover/completion requests

**Solution**:
1. Remove context-switching recompilation (files already compiled)
2. Only recompile if file changed (already tracked)
3. Cache per-file compilation state

---

### 2.4 Redundant AST Visiting - **MEDIUM** (50-150ms per edit)

**Problem**: Full AST traversal and node map rebuild on every change
- Location: `GroovyServices.visitAST()` and `ASTNodeVisitor.reset()`
- Called on: Every file change, every context switch
- Cost: Recreates all node→URI→Range mappings

**Impact**:
- On every keystroke in large file: 50-150ms rebuild
- Memory churn from thousands of Range object creations
- No delta tracking

**Solution**:
1. Implement incremental AST visiting (only affected nodes)
2. Cache Range objects permanently
3. Use structural sharing for partial updates

---

### 2.5 Gradle Build Execution - **HIGH on First Import** (30-60s)

**Problem**: Gradle "classes" task runs synchronously during initialization
- Location: `GroovyLanguageServer.java:discoverClasspath()`
- Timing: Happens on first import in background, but blocks first requests
- Cost: Full build of project classes

**Impact**:
- First request hangs while Gradle runs
- 30-60 seconds on large projects
- User sees "loading..." for minutes

**Solution**:
1. Cache discovered classpath to disk
2. Use incremental Gradle daemon (warm start)
3. Implement async classpath discovery with timeout
4. Fallback to project's `build/classes` if build fails

---

### 2.6 URI Path Normalization Overhead - **LOW** (5-10ms)

**Problem**: Repeated URI normalization in hot paths
- Location: `ASTNodeVisitor.java:getURI()` and related
- Called on: Node comparisons during lookup
- Cost: String manipulation on every comparison

**Solution**:
1. Normalize URIs once at storage time
2. Use URI.equals() directly (already normalized by Java)

---

## 3. Memory & Caching Opportunities

### 3.1 ClassGraph Result Cache (High Priority)
**Problem**: Full classpath scan recreated every import
**Solution**:
- Hash classpath entries list
- Only rescan if hash changes
- Cache scan result in memory (200MB average)
- **Estimated Savings**: 500ms-2s per import, 1-2s initialization
- **Effort**: Medium
- **Files**: `GroovyServices.java`

### 3.2 AST Range Cache (High Priority)
**Problem**: Range objects created repeatedly during lookup
**Solution**:
- Cache Range in node during AST visit phase
- Use WeakHashMap for automatic GC
- **Estimated Savings**: 100-200ms per request
- **Effort**: Low
- **Files**: `ASTNodeVisitor.java`

### 3.3 Symbol Name Index (Medium Priority)
**Problem**: Workspace symbols require O(n*m) search
**Solution**:
- Maintain `Map<String, List<ClassNode>>` during visit
- Maintain `Map<String, List<MethodNode>>` during visit
- **Estimated Savings**: 10-50ms workspace symbol queries
- **Effort**: Medium
- **Files**: `ASTNodeVisitor.java`, `WorkspaceSymbolProvider.java`

### 3.4 Import Resolution Cache (Medium Priority)
**Problem**: Import statements re-traversed per request
**Solution**:
- Cache resolved imports per source file
- Cache in FileContentsTracker alongside file contents
- **Estimated Savings**: 20-50ms per completion
- **Effort**: Low
- **Files**: `FileContentsTracker.java`, `CompletionProvider.java`

### 3.5 Method Resolution Cache (Medium Priority)
**Problem**: Method lookups walk class hierarchy repeatedly
**Solution**:
- Cache `getAllMethods(ClassNode)` with invalidation on AST rebuild
- Use TTL or generation number
- **Estimated Savings**: 50-100ms for complex hierarchies
- **Effort**: Medium
- **Files**: `GroovyASTUtils.java`, providers

---

## 4. Thread Safety & Correctness Issues

### 4.1 Lock Not Held During Provider Execution - **CRITICAL**

**Problem**: Lock released before provider future completes
- Location: `GroovyServices.java:hover()`, `completion()`, etc. (lines 226-254, 243-270)
- Risk: AST modified by concurrent edit while provider reads

**Current Code**:
```java
@Override
public CompletableFuture<Hover> hover(HoverParams params) {
    synchronized (lock) {
        URI uri = URI.create(...);
        recompileIfContextChanged(uri);
        HoverProvider provider = new HoverProvider(astVisitor);
        return provider.provideHover(...);  // Lock released here!
    }  // while provider still executing
}
```

**Risk**: `astVisitor` modified by `didChange()` while `provideHover()` reads it

**Solution**:
```java
return CompletableFuture.supplyAsync(() -> {
    synchronized (lock) {
        // ... entire provider execution under lock
        HoverProvider provider = new HoverProvider(ast Copy);
        return provider.provideHover(...);
    }
}, executorService);
```

**Impact**: Data consistency, prevents crashes
**Effort**: Low

---

### 4.2 ClassGraph Scan Timeout Missing - **MEDIUM**

**Problem**: ClassGraph.scan() can hang indefinitely
- Location: `GroovyServices.java:424`
- Risk: Server becomes unresponsive

**Solution**: Add timeout
```java
classGraphScanResult = new ClassGraph()
    .overrideClassLoaders(classLoader)
    .enableClassInfo()
    .scan(10000);  // 10 second timeout
```

**Impact**: Prevents server hangs
**Effort**: Low

---

### 4.3 FileContentsTracker Concurrent Modification - **LOW**

**Problem**: Map iteration without thread safety
- Location: `FileContentsTracker.java`
- Risk: ConcurrentModificationException rare but possible

**Solution**: Defensive copy
```java
public Set<URI> getOpenURIs() {
    return new HashSet<>(openFiles.keySet());  // Copy
}
```

**Impact**: Prevents rare crash
**Effort**: Low

---

## 5. Error Handling Gaps

### 5.1 ClassGraph Failures - **HIGH**

**Current**: Silent failure
```java
catch (ClassGraphException e) {
    classGraphScanResult = null;  // No logging, no user feedback
}
```

**Solution**: 
- Log detailed error with classpath info
- Notify user via `languageClient.showMessage()`
- Implement retry with exponential backoff

**Files**: `GroovyServices.java:424-430`

---

### 5.2 Gradle Import Failure - **HIGH**

**Current**: Logged but not visible to user
```java
catch (Exception e) {
    logger.error(e.getMessage(), e);  // Only in logs
}
```

**Solution**:
- Send diagnostic message to client
- Cache last-known-good classpath
- Implement retry mechanism

**Files**: `GroovyLanguageServer.java:discoverClasspath()`

---

### 5.3 Missing Null Checks - **MEDIUM**

**Multiple locations** where `getNodeAtLineAndColumn()` returns null but not checked downstream

**Solution**: Add defensive null checks or return empty results gracefully

---

## 6. Missing LSP Features (Lower Priority)

| Feature | Status | Effort | Impact |
|---------|--------|--------|--------|
| Code Formatting | Not implemented | High | Medium |
| Quick Fixes | Not implemented | Medium | Low |
| Folding Ranges | Not implemented | Low | Low |
| Call Hierarchy | Not implemented | Medium | Low |
| Semantic Tokens | Not implemented | Medium | Low |
| Code Lens | Not implemented | Medium | Low |
| Inlay Hints | Not implemented | Medium | Low |

---

## 7. Implementation Roadmap

### Phase 1: Critical Fixes (2-3 days)
**Goal**: Fix thread safety and error handling

1. **Fix Thread Safety** (File: `GroovyServices.java`)
   - Keep lock during provider execution
   - Use ExecutorService for async execution
   - **Effort**: 2-3 hours
   - **Impact**: Data consistency, prevents crashes

2. **Fix ClassGraph Timeout** (File: `GroovyServices.java`)
   - Add scan timeout (10 seconds)
   - **Effort**: 30 minutes
   - **Impact**: Prevents hangs

3. **Add Error Handling** (Files: `GroovyServices.java`, `GroovyLanguageServer.java`)
   - ClassGraph exception logging and user notification
   - Gradle import error notification
   - **Effort**: 2-3 hours
   - **Impact**: User visibility, debugging

4. **Test Thread Safety** (File: `src/test/`)
   - Add concurrent edit + completion test
   - Add concurrent hover + edit test
   - **Effort**: 2-3 hours

---

### Phase 2: Performance Optimization (3-5 days)
**Goal**: 2-3x speedup in response times

1. **Cache AST Ranges** (File: `ASTNodeVisitor.java`)
   - Cache Range objects in WeakHashMap during visit
   - Update node lookup to use cache
   - **Effort**: 3-4 hours
   - **Impact**: 100-200ms per request

2. **Implement ClassGraph Caching** (File: `GroovyServices.java`)
   - Hash classpath, check cache before scan
   - Make system jars optional (configurable)
   - **Effort**: 4-5 hours
   - **Impact**: 500ms-2s on classpath changes

3. **Remove Context-Switch Recompilation** (File: `GroovyServices.java`)
   - Eliminate `recompileIfContextChanged()` (files already compiled)
   - **Effort**: 2-3 hours
   - **Impact**: Eliminate 100-500ms file switch stalls

4. **Add Symbol Index** (Files: `ASTNodeVisitor.java`, providers)
   - Build class name → ClassNode map during visit
   - Build method name → MethodNode map during visit
   - **Effort**: 4-5 hours
   - **Impact**: 10-50ms workspace symbol queries (10x faster)

5. **Benchmark and Optimize** (All files)
   - Add performance logging
   - Measure before/after
   - Profile bottlenecks
   - **Effort**: 3-4 hours

---

### Phase 3: Advanced Optimizations (5-7 days)
**Goal**: Sub-100ms response times

1. **Incremental Compilation** (File: `CompilationUnitFactory.java`)
   - Compile only changed files
   - Reuse compilation state for unchanged files
   - **Effort**: 6-8 hours
   - **Impact**: File-switch stalls eliminated

2. **Spatial Index for Nodes** (File: `ASTNodeVisitor.java`)
   - Replace linear search with grid/quad-tree
   - **Effort**: 6-8 hours
   - **Impact**: 50-100ms per request (large files)

3. **Method Resolution Cache** (File: `GroovyASTUtils.java`)
   - Cache `getAllMethods()` with generation numbers
   - Invalidate on AST rebuild
   - **Effort**: 4-5 hours
   - **Impact**: 50-100ms for complex hierarchies

4. **Import Cache** (Files: `FileContentsTracker.java`, `CompletionProvider.java`)
   - Cache resolved imports per file
   - **Effort**: 3-4 hours
   - **Impact**: 20-50ms per completion

---

### Phase 4: LSP Enhancements (3-5 days, Optional)
**Goal**: Add missing features for better user experience

1. **Folding Ranges** (New file: `FoldingRangeProvider.java`)
   - Implement `textDocument/foldingRange`
   - **Effort**: 3-4 hours
   - **Impact**: Code folding in editor

2. **Quick Fixes** (New file: `CodeActionProvider.java`)
   - Add missing imports action
   - Organize imports action
   - **Effort**: 5-6 hours

---

## 8. Configuration Options

Add to properties or accept via LSP initializationOptions:

```properties
# Performance tuning
groovy.classgraph.enableSystemJars=false              # Default: false (only project jars)
groovy.classgraph.scanTimeout=10000                  # Default: 10 seconds
groovy.classgraph.cacheSize=100                      # Default: 100MB
groovy.cache.symboltable=true                        # Default: true
groovy.cache.classgraphresult=true                   # Default: true
groovy.cache.imports=true                            # Default: true
groovy.cache.methodresolution=true                   # Default: true

# Error handling
groovy.gradle.maxRetries=3                           # Default: 3 retries
groovy.gradle.retryDelayMs=1000                      # Default: 1 second
groovy.gradle.timeout=60000                          # Default: 60 seconds

# Logging
groovy.logging.profiling=false                       # Enable performance logging
groovy.logging.asyncErrors=true                      # Log async errors
```

---

## 9. Performance Targets

| Operation | Current | Target | Improvement |
|-----------|---------|--------|-------------|
| Autocomplete | 300-800ms | 50-150ms | 5-8x |
| Hover | 150-500ms | 20-50ms | 5-10x |
| Go to Definition | 200-600ms | 30-100ms | 5-8x |
| File Switch | 100-500ms stall | Negligible | 10x+ |
| Initialization | 30-60s | 5-10s | 3-6x |
| Workspace Symbols | 200-500ms | 20-50ms | 5-10x |

---

## 10. Testing Strategy

### Automated Tests
1. **Concurrency Tests** (Phase 1)
   - Concurrent edit + completion
   - Concurrent hover + edit
   - Concurrent definition + change

2. **Performance Tests** (Phase 2)
   - Benchmark each operation
   - Compare before/after optimization
   - Profile memory usage

3. **Correctness Tests** (All phases)
   - All existing tests pass
   - New edge cases covered

### Manual Testing
1. Open large Gradle project (1000+ classes)
2. Test autocomplete performance with Tab key
3. Test rapid file switching
4. Test with classpath changes (add dependency)
5. Test with broken syntax

---

## 11. Rollout Plan

1. **Branch**: Create `optimization/server-perf` branch
2. **Phase 1**: Merge after review (critical fixes)
3. **Phase 2**: Merge after testing (performance optimization)
4. **Phase 3**: Merge after profiling (advanced optimizations)
5. **Phase 4**: Optional features in future releases

---

## Success Criteria

- ✅ All operations complete in < 200ms (< 500ms on large projects)
- ✅ File switching has no noticeable delay
- ✅ Initialization completes in < 10 seconds
- ✅ Thread safety verified with stress tests
- ✅ Error handling tested and visible to users
- ✅ Memory usage stable (no leaks)
- ✅ All existing tests pass
- ✅ No functional regressions

---

## Budget Summary

| Phase | Days | Priority | Blockers |
|-------|------|----------|----------|
| Phase 1 (Critical) | 2-3 | **MUST-HAVE** | None |
| Phase 2 (Optimization) | 3-5 | **HIGH** | Phase 1 complete |
| Phase 3 (Advanced) | 5-7 | **MEDIUM** | Phase 2 working |
| Phase 4 (Features) | 3-5 | **LOW** | Optional |
| **Total** | **13-20 days** | - | - |

**Recommended Focus**: Phases 1-2 (5-8 days) for maximum user impact
