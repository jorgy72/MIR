# Google Docs Write Protocol

## Diagnosis
The issue was not that Google Drive was broken. The stable pattern is:
- document creation works,
- simple text insertion works,
- failures appear when execution becomes inconsistent across threads or when the request shape changes.

This means the main bottleneck is **consistency of execution**, not basic capability.

## Standard Write Method
Always use this sequence:

1. **Create the document**
   - `create_file`
   - MIME type: `application/vnd.google-apps.document`

2. **Capture the document ID**
   - Use the returned `fileId`
   - Do not rely on title matching if the ID is already known

3. **Write with a single plain insertion request**
   - `batch_update_document`
   - Use exact structure:

```json
{
  "document_id": "<DOC_ID>",
  "requests": [
    {
      "insertText": {
        "location": { "index": 1 },
        "text": "your text here"
      }
    }
  ]
}
```

4. **Verify immediately**
   - `get_document_text`
   - Confirm the content actually landed

## Rules
- Always write by `document_id` once it exists
- Always use `location.index = 1` for a fresh document
- Prefer one clean `insertText` request over more complex write structures at first
- After writing, always verify with a read-back step
- If the content is long, prefer smaller reliable passes over one giant insertion
- Treat each thread as its own execution context; do not assume identical behavior across threads

## Interpretation
This is a protocol-stability issue, not a core integration failure.

In project terms:
- capability is present
- execution can be noisy
- reliability improves when the pathway is standardized

## Recommended Command Pattern
For future use, treat Google Docs logging like a fixed procedure:

**Log this to N=1**
1. create or locate doc
2. write using exact `document_id + batch_update_document + insertText@index1` method
3. verify with `get_document_text`

## Practical Takeaway
The correct mindset is:
- do not improvise the write method
- use the same protocol every time
- verify every write

If uncertain, use the smallest possible test first, then scale up.
