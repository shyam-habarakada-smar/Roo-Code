# AWS Profile Authentication in Remote SSH Hosts

## Problem Statement

When using Roo-Code with AWS Bedrock provider in a remote SSH host scenario, AWS Profile authentication fails while AWS Credential mode works correctly. This happens because:

1. When VS Code connects to a remote SSH host, the extension code runs on the remote host
2. The AWS SDK's `fromIni()` function looks for AWS credentials in standard locations on the machine where the code is running (the remote host)
3. The user's AWS profiles are stored on their local machine (typically in `~/.aws/credentials` and `~/.aws/config`)
4. As a result, the AWS SDK can't find the profile on the remote host and authentication fails

## VS Code Extension Architecture

VS Code distinguishes between two kinds of extensions:

1. **UI Extensions**: 
   - Run on the user's local machine
   - Cannot directly access files in the remote workspace
   - Examples: themes, snippets, language grammars, and keymaps

2. **Workspace Extensions**:
   - Run on the same machine where the workspace is located
   - For remote workspaces, they run on the remote machine/environment
   - Can access files in the workspace
   - Examples: language servers, debuggers, and extensions that perform operations on workspace files

When a user installs an extension, VS Code automatically installs it to the correct location based on its kind. If an extension can run as either kind, VS Code will attempt to choose the optimal one for the situation.

## Key Insight

VS Code's documentation reveals that certain APIs are designed to always run on the local machine, even when called from a Workspace Extension running remotely:

> Like the clipboard API, the Webview API is always run on the user's local machine or in the browser, even when used from a Workspace Extension.

This provides a mechanism for accessing local resources from a remote extension.

## Proposed Solution: WebView-Based AWS Profile Authentication

Instead of having different authentication flows for local vs. remote scenarios, we can create a unified approach using WebViews:

1. **Always Use WebView for Profile Authentication**:
   - When AWS Profile authentication is selected, always use a WebView
   - The WebView runs locally (even in remote contexts)
   - This provides a consistent experience regardless of environment

2. **Implementation Flow**:

   ```
   ┌─────────────────────┐     ┌─────────────────────┐
   │                     │     │                     │
   │  Roo-Code Extension │     │  WebView (Local)    │
   │  (Local or Remote)  │     │                     │
   │                     │     │                     │
   └──────────┬──────────┘     └──────────┬──────────┘
              │                           │
              │ 1. Launch WebView         │
              ├──────────────────────────►│
              │                           │
              │                           │ 2. Read ~/.aws/credentials
              │                           │    and ~/.aws/config
              │                           │
              │ 3. Return credentials     │
              │◄──────────────────────────┤
              │                           │
   ┌──────────▼──────────┐                │
   │                     │                │
   │  AWS SDK            │                │
   │  (Use credentials   │                │
   │   directly)         │                │
   │                     │                │
   └─────────────────────┘                │
   ```

## Implementation Plan

1. **Create a WebView-Based Credential Provider**:
   - Implement a singleton class that manages a WebView for reading AWS credentials
   - The WebView will use browser APIs to read the local AWS credentials file
   - Credentials will be passed back to the extension via messaging

2. **Modify the AWS Bedrock Handler**:
   - Update the constructor to use the WebView-based credential provider when AWS Profile authentication is selected
   - Use the returned credentials directly instead of calling `fromIni()`
   - Include a fallback to `fromIni()` for local scenarios if the WebView approach fails

3. **Security Considerations**:
   - Ensure secure handling of credentials in the WebView
   - Implement proper error handling for cases where profiles don't exist
   - Consider adding timeout and retry mechanisms

## Benefits of This Approach

1. **Consistency**: Same authentication flow regardless of local or remote context
2. **Simplicity**: No need for redundant settings or different authentication paths
3. **Reliability**: WebViews always run locally, ensuring access to local credentials
4. **User Experience**: Transparent to the user - they just select a profile and it works
5. **Fallback Mechanism**: Still falls back to `fromIni()` as a last resort for local scenarios

## Supporting Various AWS Authentication Methods

AWS profiles support multiple authentication methods, each with unique requirements. Our WebView-based solution needs to handle all of these methods to be a complete replacement for `fromIni()`.

### AWS Profile Authentication Methods

1. **Static Credentials**: Basic access key and secret key
   ```ini
   [profile direct-auth]
   aws_access_key_id = AKIAIOSFODNN7EXAMPLE
   aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   ```

2. **Role Assumption**: Assuming another role using STS
   ```ini
   [profile role-assumption]
   role_arn = arn:aws:iam::123456789012:role/role-name
   source_profile = base-profile
   ```

3. **SSO Authentication**: Using AWS SSO
   ```ini
   [profile sso-auth]
   sso_start_url = https://my-sso-portal.awsapps.com/start
   sso_region = us-east-1
   sso_account_id = 123456789012
   sso_role_name = SSOReadOnlyRole
   ```

4. **External Process**: Using an external command
   ```ini
   [profile external-process]
   credential_process = /path/to/credential/process --arguments
   ```

5. **MFA-Based**: Requiring multi-factor authentication
   ```ini
   [profile mfa-auth]
   mfa_serial = arn:aws:iam::123456789012:mfa/user
   ```

### Enhanced WebView-Based Solution

To support all these authentication methods, we need to enhance our WebView solution:

#### Enhanced Architecture

```
┌─────────────────────┐     ┌─────────────────────────────────┐
│                     │     │                                 │
│  Roo-Code Extension │     │  Enhanced WebView (Local)       │
│  (Local or Remote)  │     │                                 │
│                     │     │  - Profile Parser               │
└──────────┬──────────┘     │  - Authentication Handler       │
           │                │  - AWS SDK Integration          │
           │                │                                 │
           │ 1. Request     │                                 │
           │    credentials │                                 │
           ├───────────────►│                                 │
           │                │                                 │
           │                │ 2. Parse profile                │
           │                │                                 │
           │                │ 3. Determine auth method        │
           │                │                                 │
           │                │ 4. Execute appropriate          │
           │                │    authentication flow          │
           │                │    (may involve browser         │
           │                │     or external process)        │
           │                │                                 │
           │ 5. Return      │                                 │
           │    credentials │                                 │
           │◄───────────────┤                                 │
           │                └─────────────────────────────────┘
```

#### Implementation Approach for Different Auth Methods

1. **Static Credentials**
   - Directly read from the credentials file
   - No additional processing needed

2. **Role Assumption**
   - Read the source profile credentials
   - Use AWS STS client in the WebView to assume the role
   - Return the temporary credentials

3. **SSO Authentication**
   - Check if SSO token cache exists and is valid
   - If not, initiate SSO authentication flow:
     - Launch a popup browser for SSO login
     - Capture and store the SSO token
   - Use the token to get SSO credentials via AWS SDK

4. **External Process**
   - Execute the credential process in the WebView using:
     - For simple cases: WebView's JavaScript capabilities
     - For complex cases: Message back to extension to run the process locally and return results

5. **MFA-Based Authentication**
   - Prompt the user for MFA code in the WebView
   - Use the code with AWS STS to get temporary credentials

#### Handling Browser-Based Authentication (SSO)

For SSO authentication that requires a browser:

1. **Option 1: In-WebView Browser**
   - Use an iframe within the WebView to handle the SSO authentication flow
   - Capture the redirect with the authentication token

2. **Option 2: External Browser with Communication Channel**
   - Launch the system browser for SSO authentication
   - Use a local server or custom protocol handler to capture the authentication result
   - Pass the result back to the WebView

3. **Option 3: Extension API Integration**
   - Use VS Code's authentication provider API if available
   - Integrate with existing authentication extensions

#### Handling External Processes

For profiles that use `credential_process`:

1. **Option 1: WebView to Extension Communication**
   - WebView sends a message to the extension
   - Extension executes the process locally
   - Result is sent back to the WebView

2. **Option 2: WebAssembly for Complex Processes**
   - For certain credential helpers, compile to WebAssembly
   - Run directly in the WebView

## Challenges and Considerations

1. **Security**:
   - Need to ensure secure handling of credentials in the WebView
   - Protect against XSS vulnerabilities
   - Secure storage of temporary tokens

2. **File Access**:
   - Need to implement file reading in the WebView (may require additional permissions)
   - Handle file system differences between operating systems

3. **User Experience**:
   - Brief WebView flash when authenticating (could be minimized with styling)
   - Seamless handling of browser-based authentication flows
   - Clear error messages for authentication failures

4. **Error Handling**:
   - Need robust error handling for cases where profiles don't exist
   - Handle network failures during authentication
   - Graceful degradation when specific authentication methods fail

5. **Authentication Method Complexity**:
   - SSO authentication requires browser interaction
   - External processes may have dependencies that are hard to satisfy in a WebView
   - Role chains (roles that assume other roles) require multiple API calls

## Interim Solution: Improved Error Detection and Messaging

While the WebView-based solution is being reviewed and implemented, we can make an immediate improvement to the user experience by enhancing error detection and messaging when AWS Profile authentication fails in remote SSH contexts.

### Implementation Approach

1. **Detect Remote SSH Context**:
   ```typescript
   function isRemoteSSH(): boolean {
     // Check if running in a remote context
     const isRemote = vscode.env.remoteName !== undefined;
     
     // Check if the remote is SSH (not DevContainers or WSL)
     const isSSH = vscode.env.remoteName === 'ssh-remote';
     
     return isRemote && isSSH;
   }
   ```

2. **Check for AWS Profile Existence**:
   ```typescript
   async function doesAwsProfileExist(profile: string): Promise<boolean> {
     try {
       // Check ~/.aws/credentials
       const credentialsPath = path.join(os.homedir(), '.aws', 'credentials');
       if (await fs.pathExists(credentialsPath)) {
         const content = await fs.readFile(credentialsPath, 'utf8');
         if (content.includes(`[${profile}]`)) {
           return true;
         }
       }
       
       // Check ~/.aws/config
       const configPath = path.join(os.homedir(), '.aws', 'config');
       if (await fs.pathExists(configPath)) {
         const content = await fs.readFile(configPath, 'utf8');
         if (content.includes(`[profile ${profile}]`)) {
           return true;
         }
       }
       
       return false;
     } catch (error) {
       // If there's an error reading the files, assume the profile doesn't exist
       return false;
     }
   }
   ```

3. **Enhance Error Messages**:
   ```typescript
   // In createMessage method
   if (error instanceof Error &&
       (error.message.includes("credentials") || error.message.includes("profile")) &&
       this.options.awsUseProfile &&
       isRemoteSSH()) {
     
     yield {
       type: "text",
       text: `Error: Could not find AWS profile "${this.options.awsProfile}" on the remote SSH host.
       
   When using VS Code with remote SSH, AWS profiles must exist on the remote machine.

   Options to resolve this issue:
   1. Create the AWS profile on the remote machine
   2. Switch to using direct AWS credentials instead of a profile
   3. Copy your AWS credentials from your local machine to the remote machine`
     };
   }
   ```

### Benefits of the Interim Solution

1. **Immediate Improvement**: Can be implemented quickly while the comprehensive solution is being developed
2. **Clear Guidance**: Provides users with actionable steps to resolve the issue
3. **Educational**: Helps users understand VS Code's remote architecture and how it affects AWS authentication
4. **Minimal Changes**: Requires only error detection and message enhancements, not architectural changes

## Next Steps

1. **Short-term**:
   - Implement the interim solution for improved error messaging
   - Test in both local and remote SSH scenarios
   - Update user documentation to explain current limitations and workarounds

2. **Long-term**:
   - Implement the enhanced WebView-based credential provider with support for all authentication methods
   - Modify the AWS Bedrock handler to use the new credential provider
   - Add comprehensive error handling and fallback mechanisms for each authentication method
   - Test in both local and remote SSH scenarios with various profile types
   - Document the new behavior for users