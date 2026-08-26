# Cross-platform command recipes

Use this reference when executing the command-heavy parts of the Pagelove
workflow. The recipes deliberately make the same HTTP requests on every
platform; only variable, temporary-file, branching, and comparison syntax
changes.

## Choose the command environment

| Operating system | Supported shell | Command name |
| --- | --- | --- |
| macOS | Bash or Zsh | `curl` |
| Linux | Bash or Zsh | `curl` |
| Windows | Windows PowerShell 5.1 or PowerShell 7+ | `curl.exe` |

Detect the operating system and active shell when possible. Do not mix examples:
backslash line continuation, `$(...)`, `case`, `/dev/null`, and `mktemp` are not
PowerShell syntax. PowerShell backticks and `$env:` variables are not POSIX shell
syntax. HTTP URLs always use `/`, including on Windows.

The recipes require curl 7.76.0 or newer because they use
`--fail-with-body`. If the capability check fails, update curl before continuing;
do not silently replace it with `--fail`, which discards the response body needed
for Pagelove error diagnosis. The curl project documents the option in the
[7.76.0 release notes](https://curl.se/ch/7.76.0.html).

### macOS and Linux prerequisite check

```bash
for required_command in curl grep sed tr mktemp cmp diff date; do
  if ! command -v "$required_command" >/dev/null 2>&1; then
    echo "$required_command is required." >&2
    exit 1
  fi
done

curl --version
if ! curl --help all | grep -q -- '--fail-with-body'; then
  echo 'curl 7.76.0 or newer is required.' >&2
  exit 1
fi
```

### Windows prerequisite check

Invoke `curl.exe` explicitly. In Windows PowerShell 5.1, `curl` is an alias for
`Invoke-WebRequest`; PowerShell 7+ removes that alias, but `curl.exe` is
unambiguous in both versions. See Microsoft's
[curl on Windows](https://learn.microsoft.com/en-us/windows/curl/) guidance.

```powershell
$CurlCommand = Get-Command curl.exe -ErrorAction SilentlyContinue
if (-not $CurlCommand) {
    throw 'curl.exe is required.'
}

curl.exe --version
$CurlHelp = curl.exe --help all
if (-not ($CurlHelp -match '--fail-with-body')) {
    throw 'curl 7.76.0 or newer is required.'
}
```

## Authenticate

Keep the API key in the current process only. Never place the literal key in a
script, repository file, shell profile, transcript, or diagnostic output. Pass
it to curl only through the expanded header variable shown below. Long-running
terminals and agents do not inherit environment changes made after they start.

### macOS and Linux

The user should inject `PAGELOVE_API_KEY` into the current process or set it in
the current shell without exposing it in command history. Then initialize the
non-secret values:

```bash
: "${PAGELOVE_API_KEY:?Set PAGELOVE_API_KEY for this shell}"
CONSOLE_URL='https://config.onpagelove.com'
KEY="Authorization: Bearer ${PAGELOVE_API_KEY}"
```

Validate the key against the authenticated console path, not the public console
origin:

```bash
AUTH_STATUS="$(curl -sS -o /dev/null -w '%{http_code}' \
  "$CONSOLE_URL/console/index.html" -H "$KEY")" || exit 1

case "$AUTH_STATUS" in
  200) ;;
  401) echo 'Pagelove rejected the API key.' >&2; exit 1 ;;
  *) echo "Unexpected console status: $AUTH_STATUS" >&2; exit 1 ;;
esac
```

### Windows

First use a value already present in the current process. If it is absent, the
following fallback loads an existing user- or machine-scoped Windows environment
variable without displaying it. It does not create or persist a new secret.

```powershell
if ([string]::IsNullOrWhiteSpace($env:PAGELOVE_API_KEY)) {
    foreach ($Scope in @('User', 'Machine')) {
        $StoredKey = [Environment]::GetEnvironmentVariable('PAGELOVE_API_KEY', $Scope)
        if (-not [string]::IsNullOrWhiteSpace($StoredKey)) {
            $env:PAGELOVE_API_KEY = $StoredKey
            break
        }
    }
}

if ([string]::IsNullOrWhiteSpace($env:PAGELOVE_API_KEY)) {
    throw 'Set PAGELOVE_API_KEY for the current process before continuing.'
}

$ConsoleUrl = 'https://config.onpagelove.com'
$KeyHeader = "Authorization: Bearer $env:PAGELOVE_API_KEY"
```

Validate it:

```powershell
$AuthStatus = curl.exe -sS -o NUL -w '%{http_code}' `
    "$ConsoleUrl/console/index.html" -H $KeyHeader
$CurlExit = $LASTEXITCODE

if ($CurlExit -ne 0) {
    throw "curl transport failure while validating the key: exit $CurlExit"
}

switch ($AuthStatus.Trim()) {
    '200' { }
    '401' { throw 'Pagelove rejected the API key.' }
    default { throw "Unexpected console status: $AuthStatus" }
}
```

Do not print `$KeyHeader`, `$env:PAGELOVE_API_KEY`, or the process environment
while troubleshooting.

## Discover hosts

Fetch the entire console page without a `Range` header and parse every
`urn:Host` block. A selector range returns only the first matching host.

### macOS and Linux

```bash
HOSTS_BODY="$(mktemp)"
HOSTS_STATUS="$(curl -sS -o "$HOSTS_BODY" -w '%{http_code}' \
  "$CONSOLE_URL/console/index.html" -H "$KEY")" || exit 1

case "$HOSTS_STATUS" in
  200) cat "$HOSTS_BODY" ;;
  *) cat "$HOSTS_BODY" >&2; exit 1 ;;
esac
```

### Windows

```powershell
$HostsBody = New-TemporaryFile
$HostsStatus = curl.exe -sS -o $HostsBody.FullName -w '%{http_code}' `
    "$ConsoleUrl/console/index.html" -H $KeyHeader
$CurlExit = $LASTEXITCODE

if ($CurlExit -ne 0) {
    throw "curl transport failure while listing hosts: exit $CurlExit"
}

if ($HostsStatus.Trim() -ne '200') {
    Get-Content -LiteralPath $HostsBody.FullName -Raw | Write-Error
    throw "Host discovery failed with HTTP $HostsStatus"
}

$HostsHtml = Get-Content -LiteralPath $HostsBody.FullName -Raw
$HostsHtml
```

Parse the complete HTML in `$HostsHtml`; do not select only the first
`itemtype="urn:Host"` element. For each candidate, retain its `hid`, `hostname`,
`org`, and exact `webdav-url` together so the public and authoring endpoints are
not mixed.

## Create a host

Create a host only when the user has selected that action. First read the
organization id, then post it to the new-host template.

### macOS and Linux

```bash
if ! ORG_BLOCK="$(curl --fail-with-body -sS \
  "$CONSOLE_URL/console/index.html" -H "$KEY" \
  -H 'Range: selector=[itemtype="urn:console:User"] [itemprop="oid"]')"; then
  printf '%s\n' "$ORG_BLOCK" >&2
  exit 1
fi

# Parse ORG_ID from the returned oid element before continuing.
ORG_ID='REPLACE_WITH_PARSED_ORG_ID'

if ! NEW_HOST_BLOCK="$(curl --fail-with-body -sS \
  "$CONSOLE_URL/console/templates/new-host.html" -X POST -H "$KEY" \
  --data-urlencode "org=${ORG_ID}")"; then
  printf '%s\n' "$NEW_HOST_BLOCK" >&2
  exit 1
fi

printf '%s\n' "$NEW_HOST_BLOCK"
```

### Windows

```powershell
$OrgBlock = curl.exe --fail-with-body -sS `
    "$ConsoleUrl/console/index.html" -H $KeyHeader `
    -H 'Range: selector=[itemtype="urn:console:User"] [itemprop="oid"]'
if ($LASTEXITCODE -ne 0) {
    $OrgBlock | Write-Error
    throw 'Unable to read the organization id.'
}

# Parse $OrgId from the returned oid element before continuing.
$OrgId = 'REPLACE_WITH_PARSED_ORG_ID'

$NewHostBlock = curl.exe --fail-with-body -sS `
    "$ConsoleUrl/console/templates/new-host.html" -X POST -H $KeyHeader `
    --data-urlencode "org=$OrgId"
if ($LASTEXITCODE -ne 0) {
    $NewHostBlock | Write-Error
    throw 'Host creation failed.'
}

$NewHostBlock
```

Parse the returned `urn:Host` block and use its exact `webdav-url`. Do not derive
an authoring URL from `hostname`.

## Deploy safely with macOS or Linux

These commands target `index.html`; substitute the intended relative remote and
local paths as needed. Confirm with the user before the first mutating request.
Upload dependencies before entry HTML.

```bash
WEBDAV='REPLACE_WITH_EXACT_ADVERTISED_WEBDAV_URL'
LOCAL_FILE='index.html'

case "$WEBDAV" in
  */) ;;
  *) WEBDAV="${WEBDAV}/" ;;
esac

INDEX_URL="${WEBDAV}index.html"
TEMP_ROOT="$(mktemp -d)"
BACKUP_FILE="${TEMP_ROOT}/index.html.before"
PROPFIND_BODY="${TEMP_ROOT}/propfind.xml"
PUT_BODY="${TEMP_ROOT}/put-response.txt"

# Read-only inventory. A successful WebDAV PROPFIND returns 207 Multi-Status.
curl --fail-with-body -sS -X PROPFIND "$WEBDAV" \
  -H "$KEY" -H 'Depth: 1'

# Determine whether the target exists and capture its current content ETag.
PROPFIND_STATUS="$(curl -sS -o "$PROPFIND_BODY" -w '%{http_code}' \
  -X PROPFIND "$INDEX_URL" -H "$KEY" -H 'Depth: 0')" || exit 1

REMOTE_EXISTS='false'
REMOTE_CONTENT_ETAG=''
case "$PROPFIND_STATUS" in
  207)
    REMOTE_EXISTS='true'
    REMOTE_CONTENT_ETAG="$(tr -d '\r\n' < "$PROPFIND_BODY" | \
      sed -n 's:.*<[^>]*getetag[^>]*>\([^<]*\)</[^>]*getetag>.*:\1:p')"
    if [ -z "$REMOTE_CONTENT_ETAG" ]; then
      echo 'The target exists but its ETag could not be read.' >&2
      exit 1
    fi
    ;;
  404) ;;
  *) cat "$PROPFIND_BODY" >&2; exit 1 ;;
esac

# Snapshot the exact version guarded by the ETag before replacing it.
if [ "$REMOTE_EXISTS" = 'true' ]; then
  BACKUP_STATUS="$(curl -sS -o "$BACKUP_FILE" -w '%{http_code}' \
    "$INDEX_URL" -H "$KEY" -H "If-Match: $REMOTE_CONTENT_ETAG")" || exit 1
  if [ "$BACKUP_STATUS" != '200' ]; then
    cat "$BACKUP_FILE" >&2
    exit 1
  fi
fi

# Example: create a directory only when it is known to be missing.
# curl --fail-with-body -sS -X MKCOL "${WEBDAV}admin/" -H "$KEY"

# Existing files require If-Match; new files require If-None-Match: *.
if [ "$REMOTE_EXISTS" = 'true' ]; then
  PUT_STATUS="$(curl -sS -o "$PUT_BODY" -w '%{http_code}' \
    -X PUT "$INDEX_URL" -H "$KEY" \
    -H "If-Match: $REMOTE_CONTENT_ETAG" \
    -H 'Content-Type: text/html' --data-binary "@${LOCAL_FILE}")" || exit 1
else
  PUT_STATUS="$(curl -sS -o "$PUT_BODY" -w '%{http_code}' \
    -X PUT "$INDEX_URL" -H "$KEY" \
    -H 'If-None-Match: *' \
    -H 'Content-Type: text/html' --data-binary "@${LOCAL_FILE}")" || exit 1
fi

case "$PUT_STATUS" in
  200|201|204) ;;
  *) cat "$PUT_BODY" >&2; exit 1 ;;
esac
```

Keep `$TEMP_ROOT` until read-back and public verification pass. If the write
returns `409` or `412`, another writer changed the target or won the create race;
stop, re-read the remote state, and obtain user direction before retrying.

## Deploy safely with PowerShell

This recipe uses only syntax available in Windows PowerShell 5.1 and PowerShell
7+. Confirm with the user before the first mutating request.

```powershell
$WebDav = 'REPLACE_WITH_EXACT_ADVERTISED_WEBDAV_URL'
$LocalFile = (Resolve-Path -LiteralPath 'index.html').Path

if (-not $WebDav.EndsWith('/')) {
    $WebDav += '/'
}

$IndexUrl = "${WebDav}index.html"
$TempRoot = Join-Path ([IO.Path]::GetTempPath()) ("pagelove-" + [guid]::NewGuid())
$null = New-Item -ItemType Directory -Path $TempRoot
$BackupFile = Join-Path $TempRoot 'index.html.before'
$PropfindBody = Join-Path $TempRoot 'propfind.xml'
$PutBody = Join-Path $TempRoot 'put-response.txt'

# Read-only inventory. A successful WebDAV PROPFIND returns 207 Multi-Status.
curl.exe --fail-with-body -sS -X PROPFIND $WebDav `
    -H $KeyHeader -H 'Depth: 1'
if ($LASTEXITCODE -ne 0) {
    throw 'Unable to inspect the WebDAV root.'
}

# Determine whether the target exists and capture its current content ETag.
$PropfindStatus = curl.exe -sS -o $PropfindBody -w '%{http_code}' `
    -X PROPFIND $IndexUrl -H $KeyHeader -H 'Depth: 0'
$CurlExit = $LASTEXITCODE
if ($CurlExit -ne 0) {
    throw "curl transport failure during PROPFIND: exit $CurlExit"
}

$RemoteExists = $false
$RemoteContentEtag = $null
switch ($PropfindStatus.Trim()) {
    '207' {
        $RemoteExists = $true
        [xml]$PropfindXml = Get-Content -LiteralPath $PropfindBody -Raw
        $EtagNode = $PropfindXml.SelectSingleNode("//*[local-name()='getetag']")
        if ($null -eq $EtagNode -or [string]::IsNullOrWhiteSpace($EtagNode.InnerText)) {
            throw 'The target exists but its ETag could not be read.'
        }
        $RemoteContentEtag = $EtagNode.InnerText
    }
    '404' { }
    default {
        Get-Content -LiteralPath $PropfindBody -Raw | Write-Error
        throw "Unexpected PROPFIND status: $PropfindStatus"
    }
}

# Snapshot the exact version guarded by the ETag before replacing it.
if ($RemoteExists) {
    $BackupStatus = curl.exe -sS -o $BackupFile -w '%{http_code}' `
        $IndexUrl -H $KeyHeader -H "If-Match: $RemoteContentEtag"
    $CurlExit = $LASTEXITCODE
    if ($CurlExit -ne 0 -or $BackupStatus.Trim() -ne '200') {
        if (Test-Path -LiteralPath $BackupFile) {
            Get-Content -LiteralPath $BackupFile -Raw | Write-Error
        }
        throw "Unable to snapshot the remote file: HTTP $BackupStatus"
    }
}

# Example: create a directory only when it is known to be missing.
# curl.exe --fail-with-body -sS -X MKCOL "${WebDav}admin/" -H $KeyHeader

$PutArgs = @(
    '-sS', '-o', $PutBody, '-w', '%{http_code}',
    '-X', 'PUT', $IndexUrl,
    '-H', $KeyHeader,
    '-H', 'Content-Type: text/html',
    '--data-binary', "@$LocalFile"
)

if ($RemoteExists) {
    $PutArgs += @('-H', "If-Match: $RemoteContentEtag")
} else {
    $PutArgs += @('-H', 'If-None-Match: *')
}

$PutStatus = curl.exe @PutArgs
$CurlExit = $LASTEXITCODE
if ($CurlExit -ne 0) {
    if (Test-Path -LiteralPath $PutBody) {
        Get-Content -LiteralPath $PutBody -Raw | Write-Error
    }
    throw "curl transport failure during PUT: exit $CurlExit"
}

switch ($PutStatus.Trim()) {
    { $_ -in @('200', '201', '204') } { }
    default {
        Get-Content -LiteralPath $PutBody -Raw | Write-Error
        throw "PUT failed with HTTP $PutStatus"
    }
}
```

Keep `$TempRoot` until read-back and public verification pass. A `409` or `412`
is not a retry signal: re-read the target and its ETag before deciding what to do.

## Verify the WebDAV read-back

Compare the complete remote file with the exact local bytes. A successful root
`PROPFIND` returns `207 Multi-Status`; treat it as success.

### macOS and Linux

Continue in the shell where `TEMP_ROOT`, `WEBDAV`, `LOCAL_FILE`, and `KEY` are
still set:

```bash
REMOTE_COPY="${TEMP_ROOT}/index.html.after"
curl --fail-with-body -sS "${WEBDAV}index.html" -H "$KEY" -o "$REMOTE_COPY"

if ! cmp -s "$LOCAL_FILE" "$REMOTE_COPY"; then
  diff -u "$LOCAL_FILE" "$REMOTE_COPY" || true
  echo 'WebDAV read-back differs from the local file.' >&2
  exit 1
fi

curl --fail-with-body -sS -X PROPFIND "$WEBDAV" \
  -H "$KEY" -H 'Depth: 1'
```

### Windows

Continue in the PowerShell process where `$TempRoot`, `$WebDav`, `$LocalFile`,
and `$KeyHeader` are still set:

```powershell
$RemoteCopy = Join-Path $TempRoot 'index.html.after'
curl.exe --fail-with-body -sS "${WebDav}index.html" `
    -H $KeyHeader -o $RemoteCopy
if ($LASTEXITCODE -ne 0) {
    throw 'Unable to read the deployed file back over WebDAV.'
}

$LocalHash = (Get-FileHash -LiteralPath $LocalFile -Algorithm SHA256).Hash
$RemoteHash = (Get-FileHash -LiteralPath $RemoteCopy -Algorithm SHA256).Hash
if ($LocalHash -ne $RemoteHash) {
    Compare-Object `
        (Get-Content -LiteralPath $LocalFile) `
        (Get-Content -LiteralPath $RemoteCopy) | Format-Table | Out-String | Write-Error
    throw 'WebDAV read-back differs from the local file.'
}

curl.exe --fail-with-body -sS -X PROPFIND $WebDav `
    -H $KeyHeader -H 'Depth: 1'
if ($LASTEXITCODE -ne 0) {
    throw 'The final WebDAV inventory failed.'
}
```

Do not dismiss a hash mismatch as a line-ending difference. `--data-binary`
uploads the file bytes present on disk, so the read-back should match those bytes.

## Verify the public application

WebDAV verification does not exercise public authorization, validation,
composition, caching, or application behavior. Use a unique query value, check
required content markers and assets, then exercise the primary interaction.

### macOS and Linux

```bash
PUBLIC='https://PUBLIC_HOSTNAME/'
CACHE_BUSTER="$(date +%s)"
PUBLIC_COPY="${TEMP_ROOT}/public-index.html"

case "$PUBLIC" in
  *\?*) PUBLIC_SEPARATOR='&' ;;
  *) PUBLIC_SEPARATOR='?' ;;
esac

curl --fail-with-body -sS \
  "${PUBLIC}${PUBLIC_SEPARATOR}cb=${CACHE_BUSTER}" -o "$PUBLIC_COPY"

# Replace with a marker that must exist in the deployed application.
grep -Fq 'REQUIRED_CONTENT_MARKER' "$PUBLIC_COPY" || {
  echo 'Required public content marker is missing.' >&2
  exit 1
}
```

### Windows

```powershell
$Public = 'https://PUBLIC_HOSTNAME/'
$UnixEpoch = [datetime]'1970-01-01T00:00:00Z'
$CacheBuster = [int64](([datetime]::UtcNow - $UnixEpoch).TotalSeconds)
$PublicCopy = Join-Path $TempRoot 'public-index.html'
$PublicSeparator = '?'
if ($Public.Contains('?')) {
    $PublicSeparator = '&'
}

curl.exe --fail-with-body -sS `
    "${Public}${PublicSeparator}cb=$CacheBuster" -o $PublicCopy
if ($LASTEXITCODE -ne 0) {
    throw 'Unable to fetch the public application.'
}

# Replace with a marker that must exist in the deployed application.
$RequiredMarker = 'REQUIRED_CONTENT_MARKER'
if (-not (Select-String -LiteralPath $PublicCopy -SimpleMatch $RequiredMarker -Quiet)) {
    throw 'Required public content marker is missing.'
}
```

Fetch every newly referenced asset through the public hostname. If an interaction
requires a signed-in browser session, have the user test it or use authorized
browser automation; a `pk_` console key is not an end-user OIDC session.

After WebDAV read-back and public verification both pass, retain the remote
snapshot only as long as rollback may still be needed, then remove the temporary
directory:

```bash
rm -rf -- "$TEMP_ROOT"
```

```powershell
Remove-Item -LiteralPath $TempRoot -Recurse -Force
```

## Update one host setting

Before changing a setting, prove that the selector returns only the intended
setting on the selected host. Never use a broad `[itemtype="urn:Host"]` mutation
unchanged when the console contains multiple hosts.

### macOS and Linux

```bash
HOST_SETTING_SELECTOR='REPLACE_WITH_SELECTOR_VERIFIED_TO_MATCH_ONE_SETTING'

curl --fail-with-body -sS "$CONSOLE_URL/console/index.html" -H "$KEY" \
  -H "Range: selector=${HOST_SETTING_SELECTOR}"

curl --fail-with-body -sS -X PUT \
  "$CONSOLE_URL/console/index.html" -H "$KEY" \
  -H "Range: selector=${HOST_SETTING_SELECTOR}" \
  -H 'Content-Type: text/html' \
  --data '<meta itemprop="default-get-authz-mode" content="deny">'
```

### Windows

```powershell
$HostSettingSelector = 'REPLACE_WITH_SELECTOR_VERIFIED_TO_MATCH_ONE_SETTING'

curl.exe --fail-with-body -sS "$ConsoleUrl/console/index.html" `
    -H $KeyHeader -H "Range: selector=$HostSettingSelector"
if ($LASTEXITCODE -ne 0) {
    throw 'The selector could not be verified.'
}

$ReplacementSetting = '<meta itemprop="default-get-authz-mode" content="deny">'
curl.exe --fail-with-body -sS -X PUT `
    "$ConsoleUrl/console/index.html" -H $KeyHeader `
    -H "Range: selector=$HostSettingSelector" `
    -H 'Content-Type: text/html' `
    --data $ReplacementSetting
if ($LASTEXITCODE -ne 0) {
    throw 'The host-setting update failed.'
}
```

Re-fetch the selected host block and verify the resulting value after any
configuration mutation.

## Troubleshooting rules shared by every platform

- A public `GET 200` does not validate the API key. Validate against
  `/console/index.html`.
- `401` from both console and WebDAV means the key or current-process environment
  is wrong. If console returns `200` but WebDAV returns `401`, re-discover the
  selected host and retry read-only `PROPFIND` against its exact `webdav-url`.
- `400` means inspect the structured error body for an invalid path or request.
- `409` or `412` means re-read the current resource and ETag. Do not blindly retry.
- `422` means the uploaded content violates a schema or constraint.
- `503` may be retried with bounded backoff and a clear stop condition.
- `507` means the allowance is exhausted; stop.
- Capture curl's process exit code separately from the HTTP status. A transport or
  TLS failure may not have a meaningful HTTP status.
- Never retry an unchanged deterministic `4xx` request on a timer.
- Never print the API key while diagnosing environment propagation.
