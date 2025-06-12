
# CircleCI Configuration Summary

This document explains how the CircleCI configuration works in this monorepo project. This project works in conjunction with the [Centralised Config](https://github.com/jenny-miggin/centralised-config) project

## 🏗️ Architecture Overview

This project uses **Dynamic Configuration** with **Path Filtering** to optimise builds in a monorepo. Instead of running all builds for every change, it only builds the packages that have actually changed.

## 📁 Project Structure

```
my-demo-monorepo/
├── packages/
│   ├── api/          # API service package
│   └── libs/         # Shared libraries package
└── .circleci/
    ├── config.yml           # Setup configuration (entry point)
    └── continue_config.yml  # Continuation configuration (actual builds)
```

## 🔄 Configuration Flow

### Phase 1: Setup & Path Detection (`config.yml`)

**Purpose**: Entry point that detects which packages have changed

**Key Features**:
- ✅ **Setup Mode**: Uses `setup: true` to enable dynamic configuration
- 🔍 **Path Filtering**: Detects changes in specific directories
- 🚀 **Continuation**: Passes control to the main build configuration

**How it works**:
1. **Checkout**: Gets the latest code
2. **Path Detection**: Uses `path-filtering` orb to detect changes:
   - `packages/api/*` → Sets `api: true` parameter
   - `packages/libs/*` → Sets `libs: true` parameter
3. **Continuation**: Passes detected changes to `continue_config.yml`

### Phase 2: Conditional Builds (`continue_config.yml`)

**Purpose**: Executes builds only for packages that have changed

**Key Features**:
- 🎯 **Conditional Workflows**: Only runs workflows for changed packages
- 🔌 **Centralised Configuration**: Delegates to external centralised configs
- 🚀 **Pipeline Triggering**: Triggers builds in the centralized config repository

**Workflow Logic**:

#### API Package Changes
```yaml
workflows:
  api:
    when: << pipeline.parameters.api >>  # Only runs if API changed
    jobs:
      - trigger-pipeline:
          orb: "https://raw.githubusercontent.com/.../frontend-team.yml"
          override-build-job: "frontend-build"
          override-deploy-job: "frontend-deploy"
```

#### Libs Package Changes
```yaml
workflows:
  libs:
    when: << pipeline.parameters.libs >>  # Only runs if libs changed
    jobs:
      - trigger-pipeline:
          orb: "https://raw.githubusercontent.com/.../libs.yml"
          override-build-job: "libs-build"
          override-deploy-job: "libs-deploy"
```

## 🎯 Key Job: `trigger-pipeline`

This job is responsible for triggering builds in the centralized configuration repository:

**What it does**:
1. **API Call**: Makes HTTP request to CircleCI API
2. **Pipeline Trigger**: Starts a new pipeline in `centralised-config` repository
3. **Parameter Passing**: Sends package-specific configuration:
   - Which orb configuration to use
   - Which build/deploy jobs to run
   - Pipeline definition ID

**API Request Structure**:
```bash
curl -X POST https://circleci.com/api/v2/project/gh/jenny-miggin/centralised-config/pipeline/run \
  --data '{
    "definition_id": "abc123",
    "parameters": {
      "orb": "https://raw.githubusercontent.com/.../frontend-team.yml",
      "override-build-job": "frontend-build",
      "override-deploy-job": "frontend-deploy"
    }
  }'
```

## 🏆 Benefits of This Approach

### ⚡ Performance Optimization
- **Faster Builds**: Only builds changed packages
- **Resource Efficiency**: Reduces unnecessary compute usage
- **Parallel Execution**: Different packages can build simultaneously

### 🔧 Centralized Management
- **DRY Principle**: Build logic lives in one centralized repository
- **Consistency**: All teams use the same build patterns
- **Easy Updates**: Change build process once, affects all projects

### 🎯 Selective Execution
- **Smart Detection**: Automatically detects which packages changed
- **Conditional Workflows**: Only runs relevant workflows
- **Scalable**: Easily add new packages without config duplication

## 🔑 Required Setup

### Environment Variables
- `CIRCLECI_API_TOKEN`: Required for triggering pipelines in the centralized config repo

### Context
- `my-context`: CircleCI context containing shared secrets and environment variables, including the above API token


## 🐛 Troubleshooting

### Common Issues

**No builds triggered despite changes**:
- Check path patterns in `mapping` section
- Verify file changes are in correct directories
- Check base revision parameter

**Pipeline trigger failures**:
- Verify `CIRCLECI_API_TOKEN` is set correctly
- Check centralized config repository permissions
- Ensure definition_id exists in target repository

**Configuration not found**:
- Verify orb URLs point to valid, accessible files
- Check branch names in URLs match actual branches
- Ensure centralized config files exist at specified paths