# Fix: Enable Autocomplete for Imported Classes from Classpath

## Problem

Autocomplete was not working for imported classes known to be on java classpath. The LSP server could parse files but failed to provide method/property suggestions for classes resolved from the project's classpath dependencies.

## Root Causes Identified

1. **Directory-based compilation failure**: The system tried to compile all directory files + open files together, causing cascading failures when any single file had errors
2. **URI format mismatch**: LSP sends URIs as `file:///` (3 slashes) but Groovy's internal parser creates `file:/` (1 slash), causing comparison failures
3. **SourceUnit naming**: Using full URIs as SourceUnit names confused Groovy's parser
4. **Missing import indexing**: Import statements weren't being visited/indexed in the AST

## Solution

### 1. Simplified Compilation Strategy (CompilationUnitFactory.java)

- **Changed**: Only compile open files, removed directory-based compilation
- **Why**: One file at a time with full classpath available is simpler and more reliable than trying to manage entire directory state
- **Impact**: Eliminates cascading failures; single file errors don't break the compilation unit

```java
// Add open files to compilation unit
fileContentsTracker.getOpenURIs().forEach(uri -> {
    String contents = fileContentsTracker.getContents(uri);
    addOpenFileToCompilationUnit(uri, contents, compilationUnit);
});
```

### 2. URI Normalization (ASTNodeVisitor.java)

- **Added**: `normalizePath()` method to convert `file:///` and `file://` to consistent `file:` format
- **Why**: Ensures reliable URI comparison between LSP protocol (3 slashes) and Groovy internals (1 slash)

### 3. SourceUnit Naming Fix (CompilationUnitFactory.java)

- **Changed**: Use `"file:" + uri.getPath()` instead of full URI as SourceUnit name
- **Why**: Full URIs caused ANTLR parser confusion; simple `file:path` format works correctly

### 4. Import Indexing (ASTNodeVisitor.java)

- **Added**: Call to `visitImports()` in `visitModule()` method
- **Why**: Ensures import statements are properly indexed for autocomplete resolution

### 5. Thread Safety (GroovyServices.java)

- **Added**: Synchronization lock around compilation and AST operations
- **Why**: Prevents race conditions when Gradle import happens concurrently with file editing

### 6. Non-blocking Gradle Import (GroovyLanguageServer.java)

- **Kept**: Async `CompletableFuture.runAsync()` for Gradle project discovery
- **Why**: Initialize response returns quickly while classpath is imported in background
- **Fixed**: Added `.get()` blocking call to launcher to ensure server stays running


## Architecture

The fix implements a clean "one file at a time" model:

1. **File opens** → tracked by FileContentsTracker
2. **Gradle classpath discovered** → async import in background  
3. **File compiles** → with full classpath via GroovyClassLoader
4. **AST visited** → imports indexed, nodes mapped to URIs
5. **Autocomplete requested** → ClassGraph scans classpath, returns matching methods

This approach is simpler, more reliable, and aligns with how most LSP servers work.

## Files Modified

- `src/main/java/net/prominic/groovyls/config/CompilationUnitFactory.java`
- `src/main/java/net/prominic/groovyls/compiler/ast/ASTNodeVisitor.java`
- `src/main/java/net/prominic/groovyls/GroovyServices.java`
- `src/main/java/net/prominic/groovyls/providers/CompletionProvider.java`
- `src/main/java/net/prominic/groovyls/GroovyLanguageServer.java`
