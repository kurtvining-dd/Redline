# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Speedrun is a userscript that enhances GitHub markdown with interactive capabilities for AWS operations and command-line building. It runs as a browser extension (via Tampermonkey/Violentmonkey) and operates on:
- GitHub repositories (github.com)
- AWS Console (console.aws.amazon.com)
- AWS SSO portals (awsapps.com)

The project transforms static documentation into executable workflows that can:
- Prompt for inputs and validate them
- Execute JavaScript code in the browser
- Retrieve AWS credentials (via federation, granted CLI, or Identity Center)
- Federate into the AWS Console with specific roles
- Build exact CLI commands with proper escaping
- Invoke AWS services (Lambda, Step Functions, EventBridge)

## Repository Structure

```
.
├── Speedrun.user.js    # Main userscript (~4000 lines, single file architecture)
├── README.md           # Project introduction with feature list
├── SECURITY.md         # Security policy for vulnerability reporting
├── cookbook/           # Example implementations and recipes
│   ├── CLI.md         # AWS CLI command building examples
│   ├── CWLInsights.md # CloudWatch Logs Insights query examples
│   ├── granted.md     # Integration with granted CLI tool
│   ├── identitycenter.md # AWS Identity Center configuration
│   └── iframes.md     # Embedding iframes in documentation
├── blog/              # Blog posts and documentation
└── misc/              # Miscellaneous content
```

## Core Architecture

### Single File Design
The entire application is contained in `Speedrun.user.js`. This is intentional for userscript distribution - there is no build process, bundler, or compilation step.

### Key Components

1. **Credentials Brokers** (lines ~355-655)
   - `SpeedrunCredentialsBroker`: Default federation via Speedrun API
   - `GrantedCredentialsBroker`: Integration with granted CLI tool
   - `IdentityCenterCredentialsBroker`: Direct AWS SSO integration
   - All brokers implement: `validate()`, `getCredentials()`, `getCacheKey()`, `getDangerKey()`

2. **Template System** (lines ~1423-1594)
   Built-in templates define output formats:
   - `federate`: Opens AWS Console with temporary credentials
   - `copy`: Copies CLI commands to clipboard with credential wrapping
   - `lambda`: Invokes Lambda functions
   - `stepfunction`: Executes Step Functions
   - `eventbridge`: Puts events on EventBridge
   - `iframe`: Embeds external content
   - `link`: Opens URLs
   - Specialized templates: `CWLInsights`, `DDBTable`, `S3Bucket`, etc.

3. **Markdown Processing**
   - Scans GitHub markdown for special syntax: `#TemplateName {params}`
   - Extracts prompts with syntax: `~~~PromptName {options}~~~`
   - Interpolates variables using JavaScript template literals
   - Supports transforms: `bashEscape()`, `toLowerCase()`, `trim()`, etc.

4. **Console Integration**
   - Detects AWS Console role/account from cookies
   - Applies custom color schemes to console tabs (role/account-based)
   - Updates favicon based on Speedrun state
   - Captures CloudWatch query time ranges for reuse

5. **Session Management**
   - Caches AWS credentials in browser storage
   - Supports multi-session mode for multiple AWS accounts
   - Stores configuration per GitHub page: `#srConfig`

### Configuration System

Each GitHub markdown page can include `#srConfig` blocks defining:
- `role`: Default IAM role to assume (prepends `speedrun-` automatically)
- `partition`: AWS partition (aws, aws-cn, aws-us-gov)
- `templates`: Custom template definitions
- `services`: Named service configurations with account/region mappings
- `logGroups`: CloudWatch log groups for queries
- `srHideUserService`: Hide personal service from dropdowns
- `srShowConfig`: Always display config

Example configuration structure:
```javascript
{
  role: 'ReadOnly',
  services: {
    MyService: {
      logGroups: 'myapp-${region}',
      regions: {
        'us-west-2': { account: 123456789012 },
        'us-east-1': { account: 123456789012 }
      }
    }
  },
  templates: {
    MyTemplate: {
      type: "copy",
      creds: true,
      value: "aws s3 ls ${bucket}"
    }
  }
}
```

## Development Commands

There are no build, test, or lint commands. This is a userscript meant to be edited directly and installed in browsers.

### Testing Changes
1. Edit `Speedrun.user.js`
2. Increment the version number (line 4: `// @version`)
3. Save the file
4. Refresh the browser tab where the userscript is installed
5. The userscript manager will auto-detect the version change and prompt to update

### Installation
Users install via userscript managers (Tampermonkey, Violentmonkey) pointing to:
- Update URL: `https://speedrun.nobackspacecrew.com/userscripts/Speedrun.meta.js`
- Download URL: `https://speedrun.nobackspacecrew.com/userscripts/Speedrun.user.js`

## Important Implementation Details

### Prompt Syntax
Prompts are embedded in markdown using `~~~` delimiters:
- `~~~Name~~~` - Simple text input
- `~~~Name {default:'value'}~~~` - With default
- `~~~Name {type:'select', options:{...}}~~~` - Dropdown
- `~~~Name {type:'textarea'}~~~` - Multi-line input
- `~~~Name {transform:'function(value)'}~~~` - With JavaScript transform

### Template Extensions
Templates can have dot-notation extensions:
- `#copy.withCreds` - Copy template with credential wrapping
- `#lambda.invoke` - Direct Lambda invocation
- Base template inherits properties, extension overrides

### Credential Caching
Credentials are cached in browser storage with keys:
- `SR:lastCreds:federate` - Last federation credentials
- `SR:lastCreds:copy` - Last CLI credentials
- `SR:sessions` - Multi-session credential cache
- Cache is flushed on AWS signin errors (HTTP 400)

### AWS Console Detection
The script detects it's running in AWS Console by checking:
- `window.location.hostname` includes `console.aws.amazon.com`
- Parses `awsuserinfo` cookie for role/account information
- Monitors URL changes via `window.onurlchange`

### CloudWatch Integration
Special handling for CloudWatch URLs:
- Decodes complex CloudWatch URL encoding schemes
- Functions: `decodeCloudWatchURIComponent()`, `unescapeCloudwatch()`, `unescapeCloudwatchLogs()`
- Extracts time ranges from CloudWatch Insights queries
- Provides "SR Timestamp" dropdown with last 5 time intervals

## Common Pitfalls

1. **Version Conflicts**: Always increment `@version` when making changes
2. **Credential Expiration**: Federation credentials typically expire after 1 hour
3. **Demo Accounts**: Accounts with negative numbers (e.g., -111111111111) are demo mode - won't attempt actual federation
4. **Role Naming**: Roles automatically get `speedrun-` prefix unless explicitly included
5. **Template Context**: Variables available in templates depend on configuration scope (page config, service config, template variables)

## External Dependencies

All dependencies are loaded via CDN (speedrun.nobackspacecrew.com):
- jQuery 3.7.1
- Lodash 4.17.21
- Select2 4.1.0-rc.0
- Day.js 1.11.13 (with UTC, duration, relativeTime plugins)
- XRegExp 5.1.2
- JSON5 2.2.3
- DOMPurify 3.2.4
- modern-screenshot 4.6.7
- srInvoke (custom library for AWS service invocation)

Do not add npm/yarn/package.json - this project has no build toolchain by design.
