# Test Picker Refactor Implementation Plan

## Executive Summary

The test picker currently shows inaccurate test counts (~111 vs actual 261) due to architectural inconsistencies between test structures and counting logic. This document outlines a pragmatic, evolutionary refactor that standardizes test metadata, improves counting accuracy, and maintains backward compatibility while respecting the working Neovim plugin architecture.

## Core Philosophy (Following CODE_STANDARDS.md)

### Evolution, Not Revolution
This refactor acknowledges pragmatic compromises necessary for a functional test system. We will document these compromises clearly and work toward better patterns over time, ensuring all functionality remains intact.

### Pre-Implementation Analysis
Before implementing, we have conducted thorough analysis to understand:
1. How metadata changes will integrate with existing test infrastructure
2. What redundancies can be eliminated without breaking test execution
3. How to improve simplicity while maintaining all test modes
4. What existing patterns and working code must be preserved
5. Where pragmatic compromises are acceptable

## Root Cause Analysis

### Current Test Architecture Problems

1. **Three Different Test Patterns**:
   - **Unit Tests**: `M.tests = { test_name = function() ... }` (works correctly)
   - **Feature/Integration Tests**: `M.run()` + internal test functions (undercounted)
   - **Command Tests**: Framework-based with `_G.himalaya_test` (special case)

2. **Counting vs Execution Mismatch**:
   - **Counting**: Static analysis of module structure
   - **Execution**: Dynamic execution with structured results
   - **Result**: Double counting, missed tests, inconsistent results

3. **Missing Test Metadata**:
   - No standard way to declare test count
   - No way to introspect complex test structures
   - No validation of counting accuracy

### Additional Context from Test Suite Analysis

4. **Test Environment Considerations**:
   - Tests run in both interactive (`:HimalayaTest`) and headless (`dev_cli.lua`) modes
   - Test isolation system requires proper cleanup and state management
   - Global test mode flags (`_G.HIMALAYA_TEST_MODE`) affect behavior

5. **Current Test Count Discrepancy**:
   - README.md states "Total Tests: 196 tests" but picker shows ~111
   - Actual execution shows 261 tests
   - This suggests multiple counting systems with different results

6. **Test Framework Integration**:
   - Existing test framework in `utils/test_framework.lua` has established patterns
   - Notification system integration requires special handling
   - CLI command mocking is essential for test isolation

## Proposed Solution: Test Metadata Standardization

### 1. Core Principle: Explicit Test Metadata

Every test module will declare its test count explicitly:

```lua
-- Standard metadata for all test modules
M.test_metadata = {
  name = "Cache System Tests",
  description = "Tests for email caching with TTL",
  count = 8,                    -- Total number of individual tests
  category = "unit",            -- unit, feature, integration, command, performance
  tags = {"cache", "storage"},  -- Optional tags for filtering
  dependencies = {"maildir"},   -- Optional dependencies
  estimated_duration_ms = 500   -- Optional performance hint
}
```

### 2. Standardized Test Interface

All test modules will implement this interface:

```lua
-- Required functions for all test modules
M.get_test_count = function()
  return M.test_metadata.count
end

M.get_test_list = function()
  -- Returns list of individual test names (for display)
  return {
    "Cache store and retrieve",
    "Cache TTL expiration",
    "Cache multiple emails",
    -- ... etc
  }
end

M.run = function()
  -- Standardized execution with consistent result format
  return {
    total = M.test_metadata.count,
    passed = passed_count,
    failed = failed_count,
    errors = error_list,
    success = failed_count == 0,
    details = test_details
  }
end
```

### 3. Direct Migration Approach (No Backward Compatibility Layer)

Instead of creating adapters that add technical debt, we will directly migrate all test files to use the new metadata system. This ensures a clean, consistent codebase:

```lua
-- Every test file will be updated to include metadata
-- No adapter layer needed - direct migration only
M.test_metadata = {
  name = "Test Suite Name",
  count = actual_count,
  category = "unit|feature|integration|command|performance"
}

-- All test files will implement the standard interface
M.get_test_count = function() return M.test_metadata.count end
M.get_test_list = function() return {...} end
```

### 4. Test Environment Preservation

The refactor preserves existing test infrastructure without compatibility layers:

- **Interactive mode**: `:HimalayaTest` command and picker interface work directly
- **Headless mode**: `dev_cli.lua` continues to function with new metadata
- **Test isolation**: Existing isolation system remains unchanged
- **Framework integration**: Direct usage of `utils/test_framework.lua`
- **Notification handling**: Existing suppression patterns preserved
- **CLI mocking**: `_G.HIMALAYA_TEST_MODE` behavior maintained

## Implementation Plan

### Testing Protocol (MANDATORY)
Following CODE_STANDARDS.md requirements, each phase MUST achieve 100% test pass rate:

```vim
" After each phase implementation
:HimalayaTest all

" Expected: All tests passing (100%)
" If any test fails, fix before proceeding
```

### Phase-Based Development Approach

Each phase follows the structured approach from CODE_STANDARDS.md:

### Phase 1: Core Infrastructure (Week 1)

**Pre-Phase Analysis**:
- [ ] Analyze test runner discovery and counting mechanisms
- [ ] Identify redundant counting logic that can be consolidated
- [ ] Document current test execution patterns
- [ ] Plan metadata integration without breaking existing tests

**Implementation Goals**:
- Create test metadata framework without technical debt
- Update test runner to use metadata directly
- Begin direct migration of test files

#### 1.1 Create Test Metadata Framework

**File**: `lua/neotex/plugins/tools/himalaya/test/utils/test_metadata.lua`

```lua
local M = {}

-- Validate test metadata structure
M.validate_metadata = function(metadata)
  local required_fields = {"name", "count", "category"}
  for _, field in ipairs(required_fields) do
    if not metadata[field] then
      error(string.format("Missing required field: %s", field))
    end
  end
  
  if type(metadata.count) ~= "number" or metadata.count < 0 then
    error("count must be a non-negative number")
  end
  
  local valid_categories = {"unit", "feature", "integration", "command", "performance"}
  if not vim.tbl_contains(valid_categories, metadata.category) then
    error(string.format("Invalid category: %s", metadata.category))
  end
  
  return true
end

-- Create standardized test interface
M.create_test_interface = function(metadata, test_functions)
  M.validate_metadata(metadata)
  
  return {
    test_metadata = metadata,
    get_test_count = function() return metadata.count end,
    get_test_list = function()
      local names = {}
      for name, _ in pairs(test_functions) do
        table.insert(names, name:gsub("^test_", ""):gsub("_", " "))
      end
      return names
    end,
    run = function()
      -- Execute tests and return standardized results
      local results = {
        total = metadata.count,
        passed = 0,
        failed = 0,
        errors = {},
        success = false,
        details = {}
      }
      
      for name, func in pairs(test_functions) do
        local ok, err = pcall(func)
        if ok then
          results.passed = results.passed + 1
        else
          results.failed = results.failed + 1
          table.insert(results.errors, {
            test = name,
            error = tostring(err)
          })
        end
      end
      
      results.success = results.failed == 0
      return results
    end
  }
end

return M
```

#### 1.2 Update Test Runner Discovery

**File**: `lua/neotex/plugins/tools/himalaya/test/test_runner.lua` (lines 846-907)

```lua
-- Enhanced test counting with metadata support (no adapters)
function M.count_test_functions_in_category(category)
  local count = 0
  
  if not M.tests[category] then
    return 0
  end
  
  for _, test_info in ipairs(M.tests[category]) do
    local success, test_module = pcall(require, test_info.module_path)
    if success then
      -- Direct metadata access - no adapters
      if test_module.test_metadata and test_module.test_metadata.count then
        count = count + test_module.test_metadata.count
      else
        -- Temporary fallback during migration
        -- TODO: Remove after all tests have metadata
        if test_module.tests and type(test_module.tests) == 'table' then
          for _, _ in pairs(test_module.tests) do
            count = count + 1
          end
        else
          count = count + 1
        end
      end
    else
      -- Module loading failed
      count = count + 1
    end
  end
  
  return count
end

-- Remove the estimate_test_count function (no longer needed)

-- Enhanced count_all_test_functions with validation
function M.count_all_test_functions()
  local total = 0
  local validation_errors = {}
  
  for category, _ in pairs(M.tests) do
    local category_count = M.count_test_functions_in_category(category)
    total = total + category_count
    
    -- Optional: Add validation to catch discrepancies
    if M.config.validate_counts then
      local actual_count = M.validate_category_count(category)
      if actual_count ~= category_count then
        table.insert(validation_errors, {
          category = category,
          expected = category_count,
          actual = actual_count
        })
      end
    end
  end
  
  -- Report validation errors if any
  if #validation_errors > 0 and not _G.HIMALAYA_TEST_MODE then
    local notify = require('neotex.util.notifications')
    notify.himalaya("Test count validation failed", notify.categories.ERROR)
  end
  
  return total
end
```

### Phase 2: Gradual Migration (Week 2-3)

#### 2.1 Migrate Unit Tests

For each unit test file, add metadata at the top:

```lua
-- Example: test_cache.lua
local M = {}

-- Add test metadata
M.test_metadata = {
  name = "Email Cache Tests",
  description = "Tests for email caching with TTL and normalization",
  count = 8,
  category = "unit",
  tags = {"cache", "storage", "ttl"},
  estimated_duration_ms = 500
}

-- Existing tests table remains unchanged
M.tests = {
  test_cache_store_and_retrieve_email = function() ... end,
  test_cache_ttl_expiration = function() ... end,
  -- ... (existing tests)
}

-- Add standardized interface
M.get_test_count = function() return M.test_metadata.count end
M.get_test_list = function()
  local names = {}
  for name, _ in pairs(M.tests) do
    table.insert(names, name:gsub("^test_", ""):gsub("_", " "))
  end
  return names
end

-- Existing run function can be preserved or updated
```

#### 2.2 Migrate Feature/Integration Tests

```lua
-- Example: test_maildir_integration.lua
local M = {}

-- Add explicit metadata
M.test_metadata = {
  name = "Maildir Integration Tests",
  description = "Comprehensive tests for complete Maildir draft system",
  count = 32, -- Explicitly declare the 32 tests this actually runs
  category = "integration",
  tags = {"maildir", "integration", "drafts"},
  estimated_duration_ms = 2000
}

-- Existing test functions remain unchanged
function M.test_foundation() ... end
function M.test_draft_manager() ... end
function M.test_composer() ... end
function M.test_integration() ... end

-- Add standardized interface
M.get_test_count = function() return M.test_metadata.count end
M.get_test_list = function()
  return {
    "Foundation Tests (8 tests)",
    "Draft Manager Tests (8 tests)",
    "Composer Tests (8 tests)",
    "Integration Tests (8 tests)"
  }
end

-- Existing run function remains unchanged
function M.run() ... end
```

#### 2.3 Update Test Runner Execution

**File**: `lua/neotex/plugins/tools/himalaya/test/test_runner.lua` (lines 465-620)

```lua
-- Enhanced test execution with direct metadata access
function M.run_test(test_info)
  -- Clear module cache
  package.loaded[test_info.module_path] = nil
  
  -- Load test module
  local ok, test_module = pcall(require, test_info.module_path)
  if not ok then
    ok, test_module = pcall(dofile, test_info.path)
  end
  
  if not ok or not test_module then
    M.results.total = M.results.total + 1
    M.results.failed = M.results.failed + 1
    table.insert(M.results.errors, {
      test = test_info.name,
      category = test_info.category,
      type = 'error',
      message = 'Failed to load test module: ' .. tostring(test_module)
    })
    return
  end
  
  -- Direct metadata validation - no adapters
  if not test_module.test_metadata then
    -- TODO: Remove this error after migration complete
    local notify = require('neotex.util.notifications')
    notify.himalaya(
      string.format("Test %s missing metadata - needs migration", test_info.name),
      notify.categories.WARNING
    )
  end
  
  -- Setup test environment
  if test_module.setup then
    pcall(test_module.setup)
  end
  
  -- Execute tests using standardized interface
  local expected_count = test_module.test_metadata and test_module.test_metadata.count or 1
  
  if test_module.run then
    local success, result = pcall(test_module.run)
    
    if success and type(result) == 'table' then
      -- Validate result structure
      local total = result.total or expected_count
      local passed = result.passed or 0
      local failed = result.failed or 0
      
      -- Validate counts match metadata
      if test_module.test_metadata and total ~= expected_count then
        if not _G.HIMALAYA_TEST_MODE then
          local notify = require('neotex.util.notifications')
          notify.himalaya(
            string.format("Test count mismatch in %s: metadata says %d, execution says %d",
              test_info.name, expected_count, total),
            notify.categories.ERROR
          )
        end
      end
      
      -- Update results
      M.results.total = M.results.total + total
      M.results.passed = M.results.passed + passed
      M.results.failed = M.results.failed + failed
      
      -- Add errors
      if result.errors then
        for _, error in ipairs(result.errors) do
          table.insert(M.results.errors, {
            test = test_info.name .. ':' .. error.test,
            category = test_info.category,
            type = 'failed',
            message = error.error or error.message
          })
        end
      end
    else
      -- Simple pass/fail result
      M.results.total = M.results.total + 1
      if success and result ~= false then
        M.results.passed = M.results.passed + 1
      else
        M.results.failed = M.results.failed + 1
        table.insert(M.results.errors, {
          test = test_info.name,
          category = test_info.category,
          type = 'failed',
          message = success and 'Test returned false' or tostring(result)
        })
      end
    end
  else
    -- No run function
    M.results.total = M.results.total + 1
    M.results.failed = M.results.failed + 1
    table.insert(M.results.errors, {
      test = test_info.name,
      category = test_info.category,
      type = 'error',
      message = 'No run function found'
    })
  end
  
  -- Teardown
  if test_module.teardown then
    pcall(test_module.teardown)
  end
end
```

### Phase 3: Enhanced Picker Interface (Week 4)

#### 3.1 Improved Picker Display

**File**: `lua/neotex/plugins/tools/himalaya/test/test_runner.lua` (lines 159-312)

```lua
-- Enhanced picker with metadata support
function M.run_with_picker(filter)
  -- ... existing filter logic ...
  
  -- Create enhanced menu structure with metadata
  local items = {}
  
  -- Count total tests with validation
  local total_tests, validation_info = M.count_all_tests_with_validation()
  
  -- Add main options with validation indicators
  local all_tests_text = string.format('Run All Tests (%d tests)', total_tests)
  if validation_info.has_estimates then
    all_tests_text = all_tests_text .. ' ~'
  end
  
  table.insert(items, { 
    text = all_tests_text, 
    value = 'all', 
    icon = '💾',
    metadata = validation_info
  })
  
  table.insert(items, { text = '                         ', value = nil, icon = '' })
  
  -- Create enhanced categories with metadata
  local categories = M.build_enhanced_categories()
  
  for _, cat in ipairs(categories) do
    local count_info = M.get_category_count_info(cat)
    local display_text = string.format('%s (%d tests)', cat.name, count_info.total)
    
    -- Add indicators for estimation vs explicit counts
    if count_info.estimated > 0 then
      display_text = display_text .. string.format(' [%d estimated]', count_info.estimated)
    end
    
    table.insert(items, {
      text = display_text,
      value = cat,
      icon = cat.icon,
      metadata = count_info
    })
  end
  
  -- ... rest of picker logic ...
end

-- Enhanced category building with metadata
function M.build_enhanced_categories()
  local categories = {
    {
      name = 'Core Data & Storage',
      icon = '💾',
      desc = 'Cache, drafts, maildir, search, templates',
      tests = {
        { category = 'unit', pattern = 'test_cache' },
        { category = 'unit', pattern = 'test_drafts' },
        { category = 'unit', pattern = 'test_maildir' },
        { category = 'unit', pattern = 'test_search' },
        { category = 'unit', pattern = 'test_templates' },
        { category = 'features', pattern = 'test_maildir_foundation' },
        { category = 'features', pattern = 'test_maildir_integration' },
      }
    },
    -- ... other categories ...
  }
  
  return categories
end

-- Get detailed count information for a category
function M.get_category_count_info(category)
  local total = 0
  local explicit = 0
  local estimated = 0
  local auto_generated = 0
  
  for _, test_spec in ipairs(category.tests) do
    if M.tests[test_spec.category] then
      for _, test_info in ipairs(M.tests[test_spec.category]) do
        if test_info.name:match(test_spec.pattern) then
          local success, test_module = pcall(require, test_info.module_path)
          if success and test_module.test_metadata then
            total = total + test_module.test_metadata.count
            
            if test_module.test_metadata.estimated then
              estimated = estimated + test_module.test_metadata.count
            else
              explicit = explicit + test_module.test_metadata.count
            end
            
            if test_module.test_metadata.auto_generated then
              auto_generated = auto_generated + 1
            end
          else
            total = total + 1
            estimated = estimated + 1
          end
        end
      end
    end
  end
  
  return {
    total = total,
    explicit = explicit,
    estimated = estimated,
    auto_generated = auto_generated
  }
end

-- Enhanced count validation
function M.count_all_tests_with_validation()
  local total = 0
  local validation_info = {
    has_estimates = false,
    auto_generated = 0,
    validation_errors = {}
  }
  
  for category, _ in pairs(M.tests) do
    local category_info = M.get_category_detailed_info(category)
    total = total + category_info.total
    
    if category_info.estimated > 0 then
      validation_info.has_estimates = true
    end
    
    validation_info.auto_generated = validation_info.auto_generated + category_info.auto_generated
    
    for _, error in ipairs(category_info.validation_errors) do
      table.insert(validation_info.validation_errors, error)
    end
  end
  
  return total, validation_info
end
```

#### 3.2 Validation and Debugging Tools

**File**: `lua/neotex/plugins/tools/himalaya/test/utils/test_validation.lua`

```lua
local M = {}

-- Validate test count accuracy by running a quick test
M.validate_test_count = function(test_module, expected_count)
  if not test_module.run then
    return false, "No run function"
  end
  
  -- Run test in validation mode
  local old_mode = _G.HIMALAYA_TEST_MODE
  _G.HIMALAYA_TEST_MODE = true
  _G.HIMALAYA_TEST_VALIDATION = true
  
  local success, result = pcall(test_module.run)
  
  _G.HIMALAYA_TEST_MODE = old_mode
  _G.HIMALAYA_TEST_VALIDATION = false
  
  if not success then
    return false, "Test execution failed: " .. tostring(result)
  end
  
  if not result or type(result) ~= 'table' then
    return false, "Invalid result structure"
  end
  
  local actual_count = result.total or 0
  if actual_count ~= expected_count then
    return false, string.format("Count mismatch: expected %d, got %d", expected_count, actual_count)
  end
  
  return true, "Validation passed"
end

-- Generate count validation report
M.generate_validation_report = function()
  local test_runner = require('neotex.plugins.tools.himalaya.test.test_runner')
  test_runner.discover_tests()
  
  local report = {
    "# Test Count Validation Report",
    "",
    string.format("Generated: %s", os.date('%Y-%m-%d %H:%M:%S')),
    ""
  }
  
  local total_tests = 0
  local total_validated = 0
  local total_errors = 0
  
  for category, tests in pairs(test_runner.tests) do
    table.insert(report, string.format("## %s Tests", category:upper()))
    table.insert(report, "")
    
    for _, test_info in ipairs(tests) do
      local success, test_module = pcall(require, test_info.module_path)
      if success and test_module.test_metadata then
        local expected = test_module.test_metadata.count
        local valid, message = M.validate_test_count(test_module, expected)
        
        total_tests = total_tests + 1
        
        if valid then
          total_validated = total_validated + 1
          table.insert(report, string.format("-  %s: %d tests", test_info.name, expected))
        else
          total_errors = total_errors + 1
          table.insert(report, string.format("- L %s: %d tests - %s", test_info.name, expected, message))
        end
      else
        total_errors = total_errors + 1
        table.insert(report, string.format("- �  %s: No metadata", test_info.name))
      end
    end
    
    table.insert(report, "")
  end
  
  -- Summary
  table.insert(report, "## Summary")
  table.insert(report, "")
  table.insert(report, string.format("- Total test modules: %d", total_tests))
  table.insert(report, string.format("- Validated: %d", total_validated))
  table.insert(report, string.format("- Errors: %d", total_errors))
  table.insert(report, string.format("- Success rate: %.1f%%", (total_validated / total_tests) * 100))
  
  return table.concat(report, '\n')
end

return M
```

### Phase 4: Migration Verification (Week 5)

#### 4.1 Add Validation Commands

**File**: `lua/neotex/plugins/tools/himalaya/commands/test_validation.lua`

```lua
local M = {}

-- Command to validate test counts
M.validate_test_counts = function()
  local validation = require('neotex.plugins.tools.himalaya.test.utils.test_validation')
  local report = validation.generate_validation_report()
  
  -- Show in floating window
  local float = require('neotex.plugins.tools.himalaya.ui.float')
  float.show('Test Count Validation', vim.split(report, '\n'))
end

-- Command to run count validation and show discrepancies
M.debug_test_counts = function()
  local test_runner = require('neotex.plugins.tools.himalaya.test.test_runner')
  test_runner.discover_tests()
  
  local discrepancies = {}
  local total_picker_count = 0
  local total_execution_count = 0
  
  for category, tests in pairs(test_runner.tests) do
    local picker_count = test_runner.count_test_functions_in_category(category)
    local execution_count = 0
    
    -- Run tests to get actual count
    for _, test_info in ipairs(tests) do
      local success, test_module = pcall(require, test_info.module_path)
      if success and test_module.run then
        local test_result = test_module.run()
        execution_count = execution_count + (test_result.total or 1)
      else
        execution_count = execution_count + 1
      end
    end
    
    total_picker_count = total_picker_count + picker_count
    total_execution_count = total_execution_count + execution_count
    
    if picker_count ~= execution_count then
      table.insert(discrepancies, {
        category = category,
        picker = picker_count,
        execution = execution_count,
        diff = execution_count - picker_count
      })
    end
  end
  
  -- Generate report
  local lines = {
    "# Test Count Discrepancy Report",
    "",
    string.format("Total Picker Count: %d", total_picker_count),
    string.format("Total Execution Count: %d", total_execution_count),
    string.format("Overall Discrepancy: %d", total_execution_count - total_picker_count),
    ""
  }
  
  if #discrepancies > 0 then
    table.insert(lines, "## Discrepancies by Category")
    table.insert(lines, "")
    
    for _, disc in ipairs(discrepancies) do
      table.insert(lines, string.format("- %s: Picker=%d, Execution=%d, Diff=%+d",
        disc.category, disc.picker, disc.execution, disc.diff))
    end
  else
    table.insert(lines, "## No discrepancies found! <�")
  end
  
  local float = require('neotex.plugins.tools.himalaya.ui.float')
  float.show('Test Count Debug', lines)
end

return M
```

#### 4.2 Add Vim Commands

**File**: `lua/neotex/plugins/tools/himalaya/init.lua` (add to existing commands)

```lua
-- Add validation commands
vim.api.nvim_create_user_command('HimalayaTestValidate', function()
  local validation = require('neotex.plugins.tools.himalaya.commands.test_validation')
  validation.validate_test_counts()
end, {
  desc = 'Validate test count accuracy'
})

vim.api.nvim_create_user_command('HimalayaTestDebug', function()
  local validation = require('neotex.plugins.tools.himalaya.commands.test_validation')
  validation.debug_test_counts()
end, {
  desc = 'Debug test count discrepancies'
})
```

## Migration Timeline (Phase-Based per CODE_STANDARDS.md)

### Phase 1: Core Infrastructure
**Timeline**: Week 1
**Test Requirement**: 100% pass rate before proceeding

1. **Pre-Phase Analysis**:
   - [ ] Document current test counting discrepancies (196 vs 111 vs 261)
   - [ ] Analyze test runner patterns and identify consolidation opportunities
   - [ ] Review existing test framework integration points
   - [ ] Plan backward compatibility strategy

2. **Implementation**:
   - [ ] Create test metadata framework
   - [ ] Update test runner to check for metadata first
   - [ ] Add temporary fallback for unmigrated tests
   - [ ] Add validation utilities

3. **Testing Protocol**:
   - [ ] Run `:HimalayaTest all` - MUST achieve 100% pass rate
   - [ ] Verify headless mode `dev_cli.lua` still works
   - [ ] Test individual test file execution
   - [ ] Document any test failures and fixes

4. **Documentation**:
   - [ ] Update this document with progress
   - [ ] Document temporary fallback approach
   - [ ] Create migration checklist for test files

5. **Commit & Review**:
   - [ ] Atomic commit: "Phase 1: Test metadata infrastructure"
   - [ ] Include list of files changed
   - [ ] Document direct migration approach

### Phase 2: Unit Test Migration
**Timeline**: Week 2
**Test Requirement**: Maintain 100% pass rate throughout

1. **Pre-Phase Analysis**:
   - [ ] Inventory all 17 unit test files
   - [ ] Analyze test count patterns
   - [ ] Identify common test structures
   - [ ] Plan incremental migration

2. **Implementation**:
   - [ ] Migrate unit tests in batches (5-6 files at a time)
   - [ ] Add explicit metadata to each file
   - [ ] Preserve existing run() functions
   - [ ] Maintain _G.HIMALAYA_TEST_RUNNER_ACTIVE compatibility

3. **Testing Protocol**:
   - [ ] Run `:HimalayaTest all` after each batch
   - [ ] Verify counts are accurate for migrated tests
   - [ ] Test both interactive and headless modes
   - [ ] No regression in test execution

4. **User Approval Gate**:
   - [ ] Request user testing after each batch
   - [ ] Address any issues found
   - [ ] Only proceed after confirmation

### Phase 3: Feature/Integration Test Migration
**Timeline**: Week 3
**Test Requirement**: Complex test structures preserved

1. **Pre-Phase Analysis**:
   - [ ] Map out complex test structures (e.g., test_maildir_integration.lua with 32 tests)
   - [ ] Document actual test counts per file
   - [ ] Plan accurate metadata representation

2. **Implementation**:
   - [ ] Migrate feature tests (8 files)
   - [ ] Migrate integration tests (4 files)
   - [ ] Add precise count metadata (not estimates)
   - [ ] Document any pragmatic compromises

3. **Testing Protocol**:
   - [ ] Run `:HimalayaTest all` - 261 tests must still execute
   - [ ] Verify picker shows accurate counts
   - [ ] Test category groupings
   - [ ] Performance test handling

### Phase 4: Picker Enhancement
**Timeline**: Week 4
**Test Requirement**: Enhanced UI without breaking changes

1. **Implementation**:
   - [ ] Update picker display logic
   - [ ] Add validation indicators
   - [ ] Improve category organization
   - [ ] Add debugging commands

2. **Testing Protocol**:
   - [ ] Test picker in interactive mode
   - [ ] Verify telescope integration (if available)
   - [ ] Test simple picker fallback
   - [ ] All counts must be accurate

### Phase 5: Validation and Documentation
**Timeline**: Week 5
**Test Requirement**: Complete system validation

1. **Validation**:
   - [ ] Run comprehensive count validation
   - [ ] Update README.md files with accurate counts
   - [ ] Remove any deprecated code (if safe)
   - [ ] Final 100% test pass verification

2. **Documentation**:
   - [ ] Update test/README.md with final count
   - [ ] Document new validation commands
   - [ ] Create troubleshooting guide
   - [ ] Archive this implementation plan

## Success Metrics

1. **Count Accuracy**: Picker shows exactly 261 tests matching execution (resolving 196 vs 111 vs 261 discrepancy)
2. **No Regressions**: All tests still pass after migration in both interactive and headless modes
3. **Improved UX**: Picker shows detailed, accurate information with validation indicators
4. **Maintainability**: Consistent pattern across all test types
5. **Backward Compatibility**: No breaking changes to existing tests, dev_cli.lua, or :HimalayaTest
6. **Test Environment Integrity**: Preserves test isolation, CLI mocking, and notification handling
7. **Documentation Consistency**: Updates README.md to reflect accurate test counts

## Benefits

1. **Immediate**: Accurate test counts in picker
2. **Short-term**: Better test organization and debugging
3. **Long-term**: Consistent test architecture for future development
4. **Maintenance**: Clear validation tools to prevent regressions

## Risk Mitigation

1. **Backward Compatibility**: Adapter layer ensures no breaking changes to existing test files, CLI, or interactive modes
2. **Gradual Migration**: Phased approach allows for incremental validation without disrupting current workflows
3. **Validation Tools**: Built-in validation prevents count drift and catches discrepancies early
4. **Test Environment Preservation**: Maintains existing test isolation, CLI mocking, and notification patterns
5. **Documentation**: Clear migration guide for future tests and updated README.md files
6. **Rollback Strategy**: Adapter layer allows reverting to original behavior if issues arise
7. **CI/CD Compatibility**: Ensures headless mode (`dev_cli.lua`) continues working with exit codes

## Architectural Patterns (Following CODE_STANDARDS.md)

### Consistent Error Handling
All test counting and metadata operations will follow established patterns:

```lua
local ok, test_module = pcall(require, test_info.module_path)
if not ok then
  logger.error('Failed to load test module', { module = test_info.module_path })
  -- Graceful fallback: count as 1 test
  return 1
end
```

### Pragmatic Compromises (Documented)
```lua
-- Temporary fallback logic exists during migration
-- This is acceptable as it will be removed after Phase 3
-- No permanent technical debt is introduced
```

### Test Structure Evolution
```lua
-- Current: Three different test patterns
-- Migration: Unified metadata while preserving execution patterns
-- Future: Consider standardizing execution patterns (post-Phase 5)
```

## Quality Checklist (Per Phase)

Before proceeding to next phase:

- [ ] `:HimalayaTest all` shows 100% pass rate
- [ ] New unit tests written for metadata framework
- [ ] Architectural compromises documented
- [ ] Working functionality preserved (interactive & headless)
- [ ] Documentation updated (FIX_PICKER.md progress)
- [ ] Atomic commit created with clear message
- [ ] Manual testing requested from user
- [ ] User approval received

## Root Cause Analysis Approach

When test counts are wrong, we will analyze root causes:

1. **Why the Discrepancy Exists**
   - Three different counting systems (README, picker, execution)
   - Inconsistent test structures (unit vs feature patterns)
   - No standardized metadata

2. **Systematic Fix**
   - Metadata standardization addresses root cause
   - Adapters handle edge cases gracefully
   - Validation tools prevent future drift

3. **Prevention**
   - Explicit metadata prevents ambiguity
   - Validation commands catch discrepancies early
   - Documentation ensures consistency

## Additional Considerations from Test Suite Analysis

### Test Mode Flag Timing
The refactor must preserve the critical timing of `_G.HIMALAYA_TEST_MODE` flag setting:
- Set before any config initialization
- Passed through all validation calls
- Respected by CLI mocking and notification systems

### Notification System Integration
The metadata system must work with existing notification categories:
- ERROR/WARNING: Always shown (unless test_validation = true)
- STATUS/BACKGROUND: Suppressed in test mode
- Validation indicators should use appropriate categories

### Multi-Mode Compatibility
The refactor must work seamlessly in:
- Interactive mode (`:HimalayaTest` with floating windows)
- Headless mode (`dev_cli.lua` with stdout output)
- Direct file execution (individual test files)

### Performance Test Considerations
Performance tests require special handling:
- Timing-sensitive operations
- Resource usage monitoring
- Regression detection

## Summary: Pragmatic Evolution

This refactor follows the CODE_STANDARDS.md philosophy of "Evolution, Not Revolution":

1. **Preserves Working Code**: All test execution paths remain functional
2. **Avoids Technical Debt**: Direct migration without adapter layers
3. **Incremental Improvement**: Phase-based approach with user approval gates
4. **Test-Driven**: 100% pass rate required at each phase
5. **Root Cause Focus**: Addresses fundamental counting inconsistencies

The refactor elegantly solves the counting problem through direct migration, improving test architecture without introducing technical debt. By migrating all tests to use explicit metadata and working incrementally, we ensure a clean, maintainable path to accurate test counting without disrupting the working Neovim plugin.