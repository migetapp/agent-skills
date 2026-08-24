# Buckets

S3-compatible object storage. A bucket sits on a resource, takes disk from its
quota, and is reached either through these endpoints or with any S3 client using
the credentials `GET /api/v1/buckets/{uuid}` returns.

Read this before creating a bucket or touching a policy. The policy and ACL
endpoints take raw S3 JSON and XML — send anything else and they answer `422`
quoting a parser error rather than a field name.

## Contents

- [What a bucket is](#buckets)
- [Endpoints](#endpoints)
- [Creating one](#create-bucket-post-apiv1buckets)
- [Walking a customer through file storage](#walking-a-customer-through-file-storage)
- [Uploading and downloading](#6-create-and-manage-a-storage-bucket)
- [Policies and ACLs](#bucket-policy--acl-ai-assisted-configuration)

## Buckets

A **Bucket** is an S3-compatible object storage container.

- Buckets are attached to a **Resource** (Miget) for quota management
- Buckets may belong to a **Project**, but the field is optional — a bucket with no project stays at workspace level. It is never inferred: omit `project_id` and the bucket is unassigned, even when the workspace has exactly one project. A bucket without a project satisfies no assignment, so it cannot be created on — or left sitting on — an assigned resource
- Buckets can be **public** or **private** visibility
- Buckets have **S3 credentials** (access key / secret key) returned in the `GET /api/v1/buckets/{uuid}` response - use these for direct S3 API access via any S3-compatible client
- Buckets support **policies** (S3-compatible JSON bucket policies)
- Buckets support **ACLs** (S3-compatible XML access control lists)
- Bucket objects can be managed via presigned upload/download URLs

## Endpoints

## Buckets (S3-Compatible Object Storage)

- `GET /api/v1/buckets` - List all buckets
- `POST /api/v1/buckets` - Create new bucket
- `GET /api/v1/buckets/{uuid}` - Get bucket details (includes S3 endpoint, S3 credentials, usage stats)
- `PUT /api/v1/buckets/{uuid}` - Update bucket (label, project, visibility, disk size)
- `DELETE /api/v1/buckets/{uuid}` - Delete bucket and all its contents
- `PUT /api/v1/buckets/{uuid}/policy` - Set or update bucket policy (S3-compatible JSON)
- `DELETE /api/v1/buckets/{uuid}/policy` - Remove bucket policy
- `PUT /api/v1/buckets/{uuid}/acl` - Set or update bucket ACL (S3-compatible XML)
- `DELETE /api/v1/buckets/{uuid}/acl` - Remove bucket ACL

## Bucket Objects (Files & Folders)

- `GET /api/v1/buckets/{uuid}/objects/list` - List objects in bucket (supports `prefix`, `limit`, `cursor`)
- `POST /api/v1/buckets/{uuid}/objects/upload_url` - Generate presigned upload URL
- `POST /api/v1/buckets/{uuid}/objects/download_url` - Generate presigned download URL
- `POST /api/v1/buckets/{uuid}/objects/create_folder` - Create a folder
- `PUT /api/v1/buckets/{uuid}/objects/{key}/rename` - Rename an object
- `DELETE /api/v1/buckets/{uuid}/objects/{key}` - Delete an object or folder

`{key}` is the full object key, percent-encoded — the slashes of a nested key
included, so `docs/report.pdf` goes in the path as `docs%2Freport.pdf`.

## Create Bucket (`POST /api/v1/buckets`)

**Required fields:**
- `label` (string) - Human-readable display name for the bucket

**Optional but important:**
- `resource_id` (string) - UUID of the compute resource to attach the bucket to (get from `GET /api/v1/resources`). Optional at the API level, but a bucket needs a resource — supply this (or the legacy alias `miget_id`, deprecated) in practice.
- `project_id` (string) - UUID of the project the bucket belongs to (get from `GET /api/v1/projects`). Never inferred: omit it and the bucket is created with no project, even in a workspace with a single project. Send `null` on `PUT /api/v1/buckets/{uuid}` to unassign an existing bucket.
- `visibility` (string) - Bucket visibility: `"public_access"` or `"private_access"` (default: `"private_access"`)
- `disk_size` (float) - Disk allocation in GiB (default: 0.1)

**Ask only what you cannot derive:**
- "What should be the bucket's display name?"
- "Which resource (Miget) should the bucket be attached to? (provide resource ID)"
- "Which project should the bucket belong to, or should it stay outside any project?"
- "Should the bucket be public or private? (default: private)"
- "How much storage do you need in GiB? (default: 0.1 GiB)"

---

## Walking a customer through file storage

1. **Ask what the files are.** User uploads, build artefacts and backups have
   different answers — a bucket suits uploads and artefacts; a shared volume
   (`addons-services.md`) suits anything the application must see as a filesystem.
2. **Pick the resource**, and tell them the disk comes out of its quota.
3. **Decide visibility deliberately.** `private_access` is the default and the
   right one unless the files are meant to be world-readable by URL.
4. **Create it, then read the credentials back** — endpoint, access key and secret
   come from `GET /api/v1/buckets/{uuid}`, not from the create response.
5. **Show them one upload end to end** with a presigned URL, so they can see the
   two-step shape: ask the API for a URL, then PUT the bytes straight to storage.

### What to suggest next

- **A bucket policy**, if anything should be publicly readable — a policy is safer
  than making the whole bucket public.
- **Object keys with a prefix per environment** (`staging/`, `production/`), which
  costs nothing now and saves a migration later.
- **A lifecycle habit**: nothing expires on its own, so old artefacts accumulate
  and keep charging disk.

## 6. Create and Manage a Storage Bucket

```http
# Step 1: Create a bucket (requires an existing resource)
POST /api/v1/buckets
{
  "label": "My Assets Bucket",
  "resource_id": "01H...resource-uuid...",
  "project_id": "01H...project-uuid...",
  "visibility": "private_access",
  "disk_size": 1.0
}

# Step 2: Get bucket details (S3 endpoint, credentials, usage)
GET /api/v1/buckets/{bucket-uuid}

# Step 3: Upload a file (get presigned URL, then upload directly to S3)
POST /api/v1/buckets/{bucket-uuid}/objects/upload_url
{
  "key": "images/logo.png",
  "size": 102400,
  "content_type": "image/png"
}
# Response contains presigned URL - upload file directly via HTTP PUT to that URL

# Step 4: List objects in bucket
GET /api/v1/buckets/{bucket-uuid}/objects/list?prefix=images/&limit=50

# Step 5: Download a file (get presigned URL)
POST /api/v1/buckets/{bucket-uuid}/objects/download_url
{
  "key": "images/logo.png"
}

# Update bucket settings
PUT /api/v1/buckets/{bucket-uuid}
{
  "label": "Updated Label",
  "visibility": "public_access",
  "disk_size": 5.0
}

# Move the bucket to another project, or send null to unassign it
PUT /api/v1/buckets/{bucket-uuid}
{
  "project_id": "01H...project-uuid..."
}

# Delete a bucket
DELETE /api/v1/buckets/{bucket-uuid}
```

## 7. Configure Bucket Access (Policy & ACL)

Bucket policies and ACLs control who can access bucket contents and how. Users rarely know the exact format - your job is to understand their intent and build the configuration for them. Policies use JSON format; ACLs use XML format. See the "Bucket Policy & ACL" section under Required Fields for templates, step-by-step guidance, and example interactions.

```http
# Set a bucket policy
PUT /api/v1/buckets/{bucket-uuid}/policy
{
  "policy": "<S3-compatible policy JSON string>"
}

# Set a bucket ACL
PUT /api/v1/buckets/{bucket-uuid}/acl
{
  "acl": "<S3-compatible ACL XML string>"
}

# Remove policy or ACL (reverts to default access rules)
DELETE /api/v1/buckets/{bucket-uuid}/policy
DELETE /api/v1/buckets/{bucket-uuid}/acl
```

## Bucket Policy & ACL (AI-Assisted Configuration)

Users rarely know S3 policy or ACL formats. Your role is to understand what they want in plain language, build the correct configuration, and send it via the API. Policies use JSON format; ACLs use XML format. Asking a user to write raw JSON/XML creates friction - instead, ask about their intent and construct the configuration yourself.

#### Step-by-step: How to handle a bucket access request

1. **Understand intent** - Ask what the user wants to achieve:
   - "Who should have access?" (everyone, specific IPs, specific users)
   - "What kind of access?" (read-only, read-write, full control)
   - "To what?" (all objects, a specific path/prefix)

2. **Choose the right mechanism:**
   - Recommend **policy** for most cases (broad rules, IP restrictions, public access)
   - Recommend **ACL** only when the user needs per-user/per-group granular S3 permissions

3. **Fetch the bucket first** - Call `GET /api/v1/buckets/{uuid}` to get the bucket `name` (needed for policy ARNs) and to check the current `policy`/`acl` state

4. **Build the configuration** - Construct JSON (for policies) or XML (for ACLs) from the templates below, substituting the actual bucket name and user-provided values

5. **Show the user what you built** - Display the formatted configuration and explain what it does before sending

6. **Send it** - `PUT /api/v1/buckets/{uuid}/policy` or `PUT /api/v1/buckets/{uuid}/acl`

#### Update Bucket Policy (`PUT /api/v1/buckets/{uuid}/policy`)

**Required fields:**
- `uuid` (string, path) - Bucket UUID
- `policy` (string) - S3-compatible bucket policy as a JSON string

**Constraints:** Must be valid JSON. Requires `buckets:operate` permission.

**Policy templates** (replace `{bucket-name}` with the actual bucket name from the GET response):

- **Public read access to all objects:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": ["s3:GetObject"],
        "Resource": ["arn:aws:s3:::{bucket-name}/*"]
      }
    ]
  }
  ```

- **Restrict access to specific IP range:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": ["arn:aws:s3:::{bucket-name}", "arn:aws:s3:::{bucket-name}/*"],
        "Condition": {
          "NotIpAddress": {
            "aws:SourceIp": "203.0.113.0/24"
          }
        }
      }
    ]
  }
  ```

- **Read-only access for a specific user:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam:::{account-id}:user/{username}"},
        "Action": ["s3:GetObject", "s3:ListBucket"],
        "Resource": ["arn:aws:s3:::{bucket-name}", "arn:aws:s3:::{bucket-name}/*"]
      }
    ]
  }
  ```

- **Allow uploads to a specific prefix:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": ["s3:PutObject"],
        "Resource": ["arn:aws:s3:::{bucket-name}/uploads/*"]
      }
    ]
  }
  ```

You can combine multiple statements in one policy. For non-standard requests, build the JSON using S3 policy conventions - `Version` is always `"2012-10-17"`, each statement needs `Effect`, `Principal`, `Action`, and `Resource`.

**Example interactions:**

```
User: "Make my bucket publicly readable"

Agent: "I'll create a policy that allows anyone to read objects from your
bucket. Let me fetch the bucket details first."
-> GET /api/v1/buckets/{uuid}  (gets bucket name: "my-assets-abc123")
-> Builds policy JSON with Resource: "arn:aws:s3:::my-assets-abc123/*"
-> Shows the user the formatted policy and explains it
-> PUT /api/v1/buckets/{uuid}/policy
```

```
User: "Only allow access from our office network 10.0.0.0/8"

Agent: "I'll set up a policy that denies all access except from your
10.0.0.0/8 network range."
-> Fetches bucket, builds deny policy with NotIpAddress condition
-> Shows and confirms, then sends PUT
```

```
User: "Remove the bucket policy"

Agent: -> DELETE /api/v1/buckets/{uuid}/policy
"Done - the bucket policy has been removed."
```

#### Update Bucket ACL (`PUT /api/v1/buckets/{uuid}/acl`)

**Required fields:**
- `uuid` (string, path) - Bucket UUID
- `acl` (string) - S3-compatible ACL as an XML string

**Constraints:** Must be valid XML. Requires `buckets:operate` permission.

**Available ACL permissions:** `READ`, `WRITE`, `READ_ACP`, `WRITE_ACP`, `FULL_CONTROL`

**ACL templates:**

- **Public read access:**
  ```xml
  <AccessControlPolicy>
    <Owner>
      <ID>{owner-canonical-id}</ID>
    </Owner>
    <AccessControlList>
      <Grant>
        <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="Group">
          <URI>http://acs.amazonaws.com/groups/global/AllUsers</URI>
        </Grantee>
        <Permission>READ</Permission>
      </Grant>
    </AccessControlList>
  </AccessControlPolicy>
  ```

- **Grant full control to another user:**
  ```xml
  <AccessControlPolicy>
    <Owner>
      <ID>{owner-canonical-id}</ID>
    </Owner>
    <AccessControlList>
      <Grant>
        <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
          <ID>{grantee-canonical-id}</ID>
        </Grantee>
        <Permission>FULL_CONTROL</Permission>
      </Grant>
    </AccessControlList>
  </AccessControlPolicy>
  ```

**Example interactions:**

```
User: "Grant read access to everyone on my assets bucket"

Agent: "I'll set an ACL granting public read access. Let me fetch the
bucket details to get the owner ID."
-> GET /api/v1/buckets/{uuid}
-> Builds ACL XML with owner ID from response
-> Shows user the formatted ACL, explains what it does
-> PUT /api/v1/buckets/{uuid}/acl
```

```
User: "I want to set up an ACL for my bucket"

Agent: "I can help with that. What access do you need to configure?
For example:
- Grant public read access to everyone
- Grant a specific user read or write access
- Grant full control to another S3 user

What would you like to set up?"
```

---
