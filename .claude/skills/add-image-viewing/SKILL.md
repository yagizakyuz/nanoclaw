---
name: add-image-viewing
description: Add image viewing support to NanoClaw. When users send images in WhatsApp, they are downloaded and saved so the agent can view them using Claude's multimodal capabilities. No external APIs required.
---

# Add Image Viewing

This skill adds automatic image download and viewing support. When users send images in WhatsApp, they're saved to the group's media folder and the agent can see them using Claude's built-in vision (the Read tool supports images).

**No external APIs or dependencies required** — Claude is natively multimodal.

## How It Works

1. User sends an image in WhatsApp (with or without caption)
2. NanoClaw downloads the image via baileys and saves it to `groups/{folder}/media/`
3. The message content includes `[Image: /workspace/group/media/img-....jpg]`
4. The agent uses the `Read` tool on that path to view the image
5. Claude processes the image and responds

## Implementation

### Step 1: Update WhatsApp Channel to Download Images

Read `src/channels/whatsapp.ts` and make these changes:

#### 1a. Add `downloadMediaMessage` import

Find the baileys import block and add `downloadMediaMessage`:

```typescript
import makeWASocket, {
  Browsers,
  DisconnectReason,
  WASocket,
  downloadMediaMessage,
  makeCacheableSignalKeyStore,
  useMultiFileAuthState,
} from '@whiskeysockets/baileys';
```

#### 1b. Add `GROUPS_DIR` to config import

Find the config import and add `GROUPS_DIR`:

```typescript
import { ASSISTANT_HAS_OWN_NUMBER, ASSISTANT_NAME, GROUPS_DIR, STORE_DIR } from '../config.js';
```

#### 1c. Add image download in the message handler

Find the `messages.upsert` handler where message content is extracted. It looks like:

```typescript
const content =
  msg.message?.conversation ||
  msg.message?.extendedTextMessage?.text ||
  msg.message?.imageMessage?.caption ||
  msg.message?.videoMessage?.caption ||
  '';
```

Replace the `const content` with `let content` and add image download logic after it:

```typescript
let content =
  msg.message?.conversation ||
  msg.message?.extendedTextMessage?.text ||
  msg.message?.imageMessage?.caption ||
  msg.message?.videoMessage?.caption ||
  '';

// Download image and save to group's media folder
if (msg.message?.imageMessage && groups[chatJid]) {
  const group = groups[chatJid];
  try {
    const buffer = await downloadMediaMessage(msg, 'buffer', {}) as Buffer;
    const mediaDir = path.join(GROUPS_DIR, group.folder, 'media');
    fs.mkdirSync(mediaDir, { recursive: true });
    const ts = new Date(Number(msg.messageTimestamp) * 1000)
      .toISOString().replace(/[:.]/g, '-');
    const filename = `img-${ts}-${msg.key.id || 'unknown'}.jpg`;
    const filePath = path.join(mediaDir, filename);
    fs.writeFileSync(filePath, buffer);
    // Append image reference so the agent can Read it (Claude is multimodal)
    // Appended (not prepended) so trigger word at start of caption still matches
    const containerPath = `/workspace/group/media/${filename}`;
    content = `${content}\n[Image: ${containerPath}]`.trim();
    logger.debug({ chatJid, filename }, 'Saved WhatsApp image');
  } catch (err) {
    logger.warn({ chatJid, err }, 'Failed to download WhatsApp image');
    content = `${content}\n[Image: failed to download]`.trim();
  }
}
```

**Important:** This only downloads images (`imageMessage`), not videos or documents.

### Step 2: Update Router to Hint About Images

Read `src/router.ts` and find the `formatMessages` function. Add an `has_image` attribute to messages containing images so the agent knows to use the Read tool:

```typescript
export function formatMessages(messages: NewMessage[]): string {
  const lines = messages.map((m) => {
    const hasImage = m.content.includes('[Image: /workspace/');
    const imageAttr = hasImage ? ' has_image="true"' : '';
    return `<message sender="${escapeXml(m.sender_name)}" time="${m.timestamp}"${imageAttr}>${escapeXml(m.content)}</message>`;
  });
  return `<messages>\n${lines.join('\n')}\n</messages>`;
}
```

### Step 3: Update Global CLAUDE.md to Instruct the Agent

Read `groups/global/CLAUDE.md` and add image viewing instructions. Without this, the agent will see `[Image: path]` in messages but **won't actually use the Read tool** — it will hallucinate descriptions instead.

#### 3a. Add image viewing to the capabilities list

Find the "What You Can Do" section and add:

```markdown
- **View images** — when a message contains `[Image: /workspace/...]`, use the `Read` tool on that path to see the image
```

#### 3b. Add an Images section before "Your Workspace"

```markdown
## Images

When a message contains `[Image: /workspace/group/media/...]`, you MUST use the `Read` tool on that exact file path to view the image before responding about it. Do NOT describe images without reading them first — you cannot see images from the text reference alone.
```

**This step is critical.** Without it the agent will make up image descriptions.

### Step 4: Build and Restart

```bash
npm run build
```

Restart the service:

- **macOS (launchd):** `launchctl kickstart -k gui/$(id -u)/com.nanoclaw`
- **Linux (systemd):** `systemctl --user restart nanoclaw`

### Step 5: Test

Tell the user:

> Image viewing is ready! Test it by:
>
> 1. Open WhatsApp on your phone
> 2. Go to a registered group chat
> 3. Send a photo (with or without a caption)
> 4. The agent will see the image and can describe it, answer questions about it, etc.
>
> Images are saved to `groups/{folder}/media/` and the agent sees them at `/workspace/group/media/` inside the container.

Watch for image downloads in the logs:

```bash
tail -f logs/nanoclaw.log | grep -i "image\|media"
```

---

## How the Agent Sees Images

When you send a photo with the caption "what is this?", the agent receives:

```xml
<message sender="User" time="2026-02-18T..." has_image="true">what is this?
[Image: /workspace/group/media/img-2026-02-18T12-00-00-000Z-ABC123.jpg]</message>
```

The agent then uses the `Read` tool on the image path. Claude's multimodal capabilities let it see and understand the image content.

---

## Storage

Images are stored in `groups/{folder}/media/`. To manage disk usage:

```bash
# Check total size of saved images
du -sh groups/*/media/

# Clean up images older than 30 days
find groups/*/media/ -name "img-*.jpg" -mtime +30 -delete
```

You can also ask the agent to set up a scheduled task to clean old images automatically.

---

## Troubleshooting

### Agent doesn't respond to images

- Check logs for download errors: `grep -i "image\|media" logs/nanoclaw.log | tail -20`
- Verify the media directory exists: `ls groups/{folder}/media/`
- Ensure the group folder is mounted (it is by default for registered groups)

### Agent describes images incorrectly (hallucinating)

The agent sees `[Image: path]` in the message but isn't actually reading the file — it's making up descriptions. This means the global CLAUDE.md is missing the image instructions from Step 3. Add the `## Images` section telling the agent it MUST use the `Read` tool on image paths.

### "Failed to download WhatsApp image"

- WhatsApp media links expire after some time. If the message is old, the download may fail.
- Check network connectivity from the host.
- Restart WhatsApp auth if downloads consistently fail: `npm run auth`

### Images not appearing in container

The group folder is mounted at `/workspace/group/` inside the container, so `groups/{folder}/media/` maps to `/workspace/group/media/`. Verify:

```bash
docker run --rm -v $(pwd)/groups/main:/workspace/group --entrypoint ls nanoclaw-agent:latest /workspace/group/media/
```

---

## Removing Image Viewing

To remove the feature:

1. Revert changes in `src/channels/whatsapp.ts`:
   - Remove `downloadMediaMessage` from imports
   - Remove `GROUPS_DIR` from config import
   - Remove the image download block
   - Change `let content` back to `const content`

2. Revert changes in `src/router.ts`:
   - Remove the `has_image` attribute logic

3. Revert changes in `groups/global/CLAUDE.md`:
   - Remove the "View images" capability line
   - Remove the "Images" section

4. Rebuild:
   ```bash
   npm run build
   systemctl --user restart nanoclaw  # or launchctl kickstart on macOS
   ```

5. Optionally clean up saved images:
   ```bash
   rm -rf groups/*/media/
   ```
