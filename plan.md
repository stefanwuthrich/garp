# Performance and Feature Improvement Plan for garp

## Executive Summary

garp is a high-performance, pure-Go document search tool with a TUI interface. After thorough analysis of the codebase, this plan identifies key opportunities for performance optimization and feature enhancements while maintaining the tool's core philosophy: zero external dependencies, pure Go implementation, and excellent user experience.

## Current Architecture Analysis

### Strengths
1. **Pure Go Implementation**: No external dependencies (ripgrep, etc.)
2. **Concurrent Processing**: Multi-core parallel processing with worker pools
3. **Smart Prefiltering**: Streaming prefilters for binary formats reduce unnecessary extraction
4. **Memory Optimization**: Size-aware file reading with caps for large files
5. **Beautiful TUI**: Live progress updates with bubbletea/lipgloss

### Performance Bottlenecks Identified

#### 1. **Regex Compilation Inefficiency** (High Priority)
- **Location**: `search/filter.go`, `search/cleaner.go`
- **Issue**: Regexes are compiled repeatedly in hot paths
- **Impact**: Significant CPU overhead, especially for large file sets
- **Evidence**:
  - `buildWordRegexLower()` and `buildWordRegexCI()` compile new regex on every call
  - Called inside loops in `CheckTextContainsAllWords()`, `StreamContainsAllWordsDecidedWithCap()`
  - Pattern matching in `ExtractMeaningfulExcerpts()` recompiles per term

#### 2. **String Allocation Overhead** (High Priority)
- **Location**: Throughout `search/filter.go` and `search/cleaner.go`
- **Issue**: Excessive string concatenation and slice operations
- **Impact**: Memory pressure and GC overhead
- **Examples**:
  - `strings.ToLower(text)` creates full copy of text in `CheckTextContainsAllWords()`
  - Multiple `strings.ReplaceAll()` chains in `CleanContent()`
  - String builder usage is inconsistent

#### 3. **File I/O Efficiency** (Medium Priority)
- **Location**: `search/filter.go` - file reading operations
- **Issue**: Sequential reads without optimal buffer sizes
- **Impact**: I/O wait time, especially for network-mounted filesystems
- **Details**:
  - Fixed 64KB chunk size may not be optimal for all scenarios
  - No read-ahead hints to OS beyond `FADV_DONTNEED`

#### 4. **PDF Processing Bottleneck** (Medium Priority)
- **Location**: `search/pdf/simple.go`, `search/engine.go`
- **Issue**: Complex PDF extraction with temporary directories
- **Impact**: High latency for PDF-heavy searches
- **Details**:
  - Creates temp directories for each batch extraction
  - Single-threaded PDF token with 250ms timeout is conservative
  - Governor pacing may be too restrictive for modern systems

#### 5. **Duplicate Work in Multi-Word Searches** (Medium Priority)
- **Location**: `search/engine.go`, `FilterCandidates()`
- **Issue**: File content may be read multiple times
- **Impact**: Redundant I/O and CPU for each extraction attempt
- **Example**: Binary files read for prefilter, then extracted, then checked for excludes

### Feature Enhancement Opportunities

#### 1. **Search Result Caching** (High Value)
- **Benefit**: Dramatically faster repeated searches in same directory
- **Implementation**: LRU cache of file hash → extracted content
- **Considerations**: Memory limits, cache invalidation on file changes

#### 2. **Incremental Search** (High Value)
- **Benefit**: Refine searches without re-scanning
- **Implementation**: Cache initial scan results, filter in memory
- **Use Case**: User starts with broad terms, then adds more constraints

#### 3. **Index Mode** (High Value)
- **Benefit**: Near-instant searches for frequently-searched directories
- **Implementation**: Pre-build inverted index for document corpus
- **Trade-off**: Disk space vs. search speed

#### 4. **Parallel PDF Processing** (Medium Value)
- **Current**: Concurrency=2 with strict timeouts
- **Improvement**: Adaptive concurrency based on system resources
- **Benefit**: Better utilization of multi-core systems

#### 5. **Smarter Worker Pool Sizing** (Medium Value)
- **Current**: Fixed or CLI-specified worker counts
- **Improvement**: Runtime adaptation based on file types and system load
- **Benefit**: Better resource utilization, fewer context switches

#### 6. **Fuzzy Matching** (Medium Value)
- **Benefit**: Find documents with typos or variants
- **Implementation**: Levenshtein distance or trigram matching
- **Use Case**: "urgnet" finds "urgent"

#### 7. **Search History** (Low Value, High UX)
- **Benefit**: Quick access to previous searches
- **Implementation**: Simple JSON file in user config directory
- **UX**: Arrow-key history navigation like shell

#### 8. **Export Results** (Low Value)
- **Benefit**: Share or process results externally
- **Implementation**: JSON/CSV output format
- **Use Case**: Batch processing, reporting

## Detailed Performance Improvement Plan

### Phase 1: Quick Wins (1-2 days)

#### 1.1 Regex Compilation Caching
**Files**: `search/filter.go`, `search/cleaner.go`

**Changes**:
```go
// Add package-level cache with bounded size
var regexCache sync.Map // map[string]*regexp.Regexp
var regexCacheSize atomic.Int64
const maxRegexCacheEntries = 1000

func getCachedRegex(pattern string) *regexp.Regexp {
    if re, ok := regexCache.Load(pattern); ok {
        return re.(*regexp.Regexp)
    }
    re := regexp.MustCompile(pattern)
    
    // Bound cache size to prevent memory leaks
    if regexCacheSize.Load() < maxRegexCacheEntries {
        regexCache.Store(pattern, re)
        regexCacheSize.Add(1)
    }
    return re
}
```

**Expected Impact**: 15-30% reduction in CPU time for searches with multiple terms

#### 1.2 String Buffer Optimization
**Files**: `search/cleaner.go`, `search/filter.go`

**Changes**:
- Replace `strings.ToLower()` with case-insensitive regex where possible
- Use `strings.Builder` consistently
- Pre-allocate buffers with estimated capacity
- Reuse buffers via sync.Pool for hot paths

**Expected Impact**: 10-20% reduction in memory allocations, faster GC

#### 1.3 Binary Format Detection Optimization
**Files**: `search/filter.go`

**Changes**:
- Cache extension → format map at startup
- Use map lookup instead of repeated `filepath.Ext()` calls
- Pre-compute lowercase extensions

**Expected Impact**: 5-10% improvement in file discovery phase

### Phase 2: Architectural Improvements (3-5 days)

#### 2.1 Content Caching Layer
**New File**: `search/cache.go`

**Features**:
- LRU cache with configurable size (default 100MB)
- Key: file path + modification time
- Value: extracted clean content
- Thread-safe with read-write locks
- Metrics: hit rate, evictions

**Expected Impact**: 50-80% faster repeat searches, 10-20% faster initial searches (due to dedup)

#### 2.2 Adaptive Worker Pool
**Files**: `search/engine.go`

**Changes**:
- Monitor queue depth and adjust workers dynamically
- Separate pools for light (text) vs heavy (binary) files
- Work-stealing between pools when idle
- CPU affinity hints for workers

**Expected Impact**: 15-25% better CPU utilization, reduced latency variance

#### 2.3 Streaming Content Processor
**New File**: `search/streaming.go`

**Features**:
- Pipeline architecture: read → prefilter → extract → match → result
- Backpressure handling to prevent memory bloat
- Early termination when result limit reached
- Zero-copy where possible

**Expected Impact**: 20-30% reduction in memory usage, better responsiveness

### Phase 3: Advanced Features (5-7 days)

#### 3.1 Indexing Mode
**New Files**: `search/index.go`, `search/index_builder.go`

**Features**:
- Build inverted index: term → [file positions]
- Compressed on-disk storage (gzip)
- Incremental updates (watch file system)
- Query optimization with index statistics

**Expected Impact**: Sub-second searches on indexed corpora (100x+ speedup)

#### 3.2 Fuzzy Matching
**New File**: `search/fuzzy.go`

**Implementation**:
- Trigram-based similarity (good balance of speed/accuracy)
- Configurable threshold via `--fuzzy N` (0=exact, 100=very loose)
- Works with existing distance window logic
- Fallback to exact match for performance

**Expected Impact**: Better user experience, finds 10-30% more relevant results

#### 3.3 Smart Binary Format Handlers
**Files**: `search/extractor.go`, individual extractors

**Improvements**:
- Streaming extraction for large DOC/DOCX/ODT files
- Selective XML parsing (skip non-text elements)
- Parallel sheet processing for XLSX
- OCR integration for image-based PDFs (opt-in)

**Expected Impact**: 2-5x faster extraction for complex Office documents

## Implementation Priorities

### Critical Path (Must-Have)
1. Regex compilation caching ⭐⭐⭐
2. String buffer optimization ⭐⭐⭐
3. Content caching layer ⭐⭐

### High Value (Should-Have)
4. Adaptive worker pool ⭐⭐
5. Binary format detection optimization ⭐⭐
6. Streaming content processor ⭐

### Nice to Have
7. Indexing mode
8. Fuzzy matching
9. Search history
10. Export results

## Performance Benchmarking Plan

### Test Corpus
- 10,000 mixed documents (text, PDF, Office)
- Size range: 1KB - 10MB
- Term frequency: rare, medium, common
- Directory depth: flat, shallow, deep

### Metrics to Track
1. **Throughput**: Files processed per second
2. **Latency**: Time to first result
3. **Memory**: Peak RSS, allocations per file
4. **CPU**: User time, system time, context switches
5. **I/O**: Read bandwidth, IOPS, cache hit rate

### Baseline vs. Optimized

**Note**: Baseline values should be measured on a reference system (e.g., 8-core CPU, 16GB RAM, SSD) using the standard test corpus before optimization work begins. These are estimated targets based on code analysis:

```
Metric                  Baseline    Target      Stretch    Measurement Method
--------------------------------------------------------------------------------
Files/sec (text)        ~500        750         1000       `time garp term --code` on 10K files
Files/sec (binary)      ~50         100         150        `time garp term` on 1K PDFs/docs
Time to first result    ~2s         0.5s        0.1s       Measured from invocation to first hit
Peak memory (10K files) ~500MB      300MB       200MB      `/usr/bin/time -v` or `pprof`
CPU efficiency          ~60%        80%         90%        CPU time / (wall time × cores)
Cache hit rate          0%          40%         60%        Instrumented cache metrics
```

**Actual baselines must be established** via benchmarking before claiming specific improvements.

## Testing Strategy

### Unit Tests (Currently Missing)
**Priority**: Critical

Create test files:
- `search/filter_test.go`: File discovery, prefiltering
- `search/engine_test.go`: Search orchestration, worker pools
- `search/cleaner_test.go`: Content cleaning, excerpt extraction
- `search/extractor_test.go`: Binary format extraction
- `search/cache_test.go`: Cache operations (new)

**Coverage Goal**: 70%+ for core search logic

### Integration Tests
**Priority**: High

Test scenarios:
- End-to-end search on known corpus
- Multi-word AND/OR logic correctness
- Exclude filters
- Edge cases: empty files, huge files, binary junk
- Concurrency: race conditions, deadlocks

### Performance Tests
**Priority**: High

Benchmark suites:
- `search/bench_test.go`: Regex caching, string ops, prefilters
- `search/bench_integration_test.go`: Full search scenarios
- Memory profiling: `go test -memprofile`
- CPU profiling: `go test -cpuprofile`

### Regression Tests
**Priority**: Medium

Ensure optimizations don't break:
- Exact search results match (byte-for-byte)
- UI rendering stays consistent
- Error handling preserved
- Edge case behavior unchanged

## Risk Mitigation

### Performance Risks
1. **Cache thrashing**: Mitigate with LRU eviction + size limits
2. **Memory bloat**: Mitigate with streaming + backpressure
3. **Regex cache growth**: Mitigate with bounded cache (1000 entries max)

### Correctness Risks
1. **False negatives**: Prevent with comprehensive test suite
2. **False positives**: Validate with integration tests
3. **Race conditions**: Use Go race detector in CI

### Compatibility Risks
1. **Breaking changes**: Version APIs, deprecate gracefully
2. **Platform differences**: Test on Linux, macOS, Windows
3. **Dependency updates**: Pin versions, test upgrades

## Monitoring & Observability

### Metrics to Expose (Optional)
- Search latency percentiles (p50, p95, p99)
- Files processed counters by type
- Cache hit/miss rates
- Worker pool utilization
- Error rates by type

### Implementation
- Prometheus-compatible metrics endpoint (opt-in via `--metrics :9090`)
- Structured logging with levels (use existing TUI output)
- Profile endpoints for live debugging (`pprof`)

## Documentation Updates

### README.md
- Add performance characteristics section
- Document new flags (--cache-size, --fuzzy, --index)
- Update benchmarks with latest numbers

### Architecture Documentation
- Create `ARCHITECTURE.md` with system diagram
- Document worker pool design
- Explain caching strategy
- Index file format specification

### Contributing Guide
- Benchmarking requirements for PRs
- Performance regression policy
- Code review checklist

## Rollout Plan

### Version 0.6 (Performance Release)
- Regex caching
- String buffer optimization
- Binary format detection optimization
- Comprehensive test suite
- Updated benchmarks

### Version 0.7 (Features Release)
- Content caching layer
- Adaptive worker pool
- Fuzzy matching (opt-in)
- Search history

### Version 0.8 (Advanced Release)
- Indexing mode
- Streaming architecture
- Smart binary handlers
- Export capabilities

## Success Criteria

### Performance Goals
✓ 2x improvement in text file throughput
✓ 3x improvement in binary file throughput  
✓ 50% reduction in memory usage
✓ Sub-second time to first result on typical searches

### Quality Goals
✓ 70%+ test coverage on core modules
✓ Zero known correctness regressions
✓ All benchmarks pass on CI
✓ No data races detected

### User Experience Goals
✓ Faster perceived performance (time to first result)
✓ Predictable resource usage
✓ Better error messages
✓ Smooth TUI with no stuttering

## Maintenance & Evolution

### Long-term Vision
- Distributed search across machines (garp cluster)
- Machine learning for relevance ranking
- Support for more file formats (archives, databases)
- Plugin system for custom extractors
- REST API for programmatic access

### Technical Debt to Address
1. Missing test coverage (critical)
2. Hard-coded constants (make configurable)
3. Error handling inconsistencies (standardize)
4. Magic numbers in timeouts (document/configure)
5. Incomplete `--only` type filtering (implement)

### Community & Ecosystem
- Performance tuning guide for users
- Extractor development guide for contributors
- Blog post: "Building a fast document search in Go"
- Conference talk submission

## Conclusion

This plan provides a comprehensive roadmap for enhancing garp's performance and features while maintaining its core strengths: pure Go, zero dependencies, and excellent UX. The phased approach allows for incremental delivery of value, with quick wins in Phase 1 and strategic improvements in later phases.

**Estimated Total Effort**: 

- **Phase 1 (Quick Wins)**: 2-3 days development + 1 day testing = **3-4 days**
- **Phase 2 (Architecture)**: 4-6 days development + 2 days testing/integration = **6-8 days**  
- **Phase 3 (Advanced)**: 6-8 days development + 2-3 days testing = **8-11 days**
- **Documentation & Polish**: 2-3 days across all phases
- **Total**: **19-26 days** of focused development (add 20-30% contingency for integration challenges)

**Realistic Schedule**: 4-5 weeks with one developer, or 2-3 weeks with two developers working in parallel.

**Expected Overall Impact**: 2-3x performance improvement on typical workloads, 100x+ with indexing, significantly enhanced user experience, and a solid foundation for future growth.

---

*This plan was created through comprehensive analysis of the garp codebase on 2025-11-03.*
