---
name: reels-maker
description: "Crops and creates a short Instagram Reel (max 6 seconds) from a source folder. Verifies CloudGlue + Hyperframes setup, uploads source videos to Google Drive, analyzes each video with CloudGlue, selects best segments on theme, cuts/joins with ffmpeg, overlays hook text with Hyperframes HTML composition + CLI render, saves directly to project folder."
---

# Reels Maker

Creates a punchy 6-second Instagram Reel: videos go to Google Drive, CloudGlue understands each one and gives timecodes, you pick the best moments, ffmpeg cuts and joins them, Hyperframes overlays the hook with styled text and renders the final video directly into the project folder.

---

## Жёсткие правила — не нарушать

1. **Только Hyperframes для текстового оверлея.** Для наложения хука на видео используй исключительно Hyperframes (HTML-композиция + `npx hyperframes render`). Никакого ffmpeg `drawtext`, PIL/Pillow или других инструментов для текста.

2. **ffmpeg — только для нарезки и склейки.** Не используй ffmpeg для текста или финального рендера с хуком.

3. **Если что-то не работает — остановись и реши проблему.** Не ищи обходных путей. Сообщи пользователю о проблеме и что именно ты делаешь для её решения.

---

## Task list rules

**Create all steps as a task list upfront — not one at a time.**

At the very start:
1. `TaskCreate` all 6 steps at once (all `pending`, except the first one which is set to `in_progress`)

At the start of each step:
1. Mark the current step `in_progress` (if not already)
2. Do the work
3. `TaskUpdate` → `completed`
4. Move on to the next step in the list (mark it `in_progress`)

This keeps the full task list visible from the start and shows progress as steps complete.

---

## Step 1 — Verify environment (CloudGlue + Hyperframes)

**→ TaskUpdate Step 1:** `in_progress`

### 1a — CloudGlue

Call `Cloudglue:list_videos`. Interpret the result:

| Result | Meaning |
|--------|--------|
| Returns list (empty or with videos) | ✅ CloudGlue API configured, proceed |
| `Command failed with no output` | ❌ API key missing or wrong |

**If ❌:** stop and tell the user:
> CloudGlue API key not configured. Go to [app.cloudglue.dev/home/api-keys](https://app.cloudglue.dev/home/api-keys), copy your key, then add it to Claude Desktop's config (`~/Library/Application Support/Claude/claude_desktop_config.json`) and restart Claude Desktop.

Do not continue until `list_videos` succeeds.

### 1b — Hyperframes CLI

Check that Hyperframes is available on the device:

```bash
npx hyperframes --version 2>/dev/null && echo "ok" || echo "missing"
```

**If missing — install:**
```bash
npm install -g @hyperframes/cli 2>&1 | tail -3
npx hyperframes --version
```

**If install fails** (no network): tell the user:
> Hyperframes CLI not found. Please run in your terminal:
> ```
> npm install -g @hyperframes/cli
> ```
> Then write «ready» and I'll continue.

Do not continue until `npx hyperframes --version` succeeds.

**→ TaskUpdate Step 1:** `completed`

---

## Step 2 — Убедиться что видео есть в Google Drive (папка `claude_for_reels`)

**→ TaskUpdate Step 2:** `in_progress`

CloudGlue анализирует видео через `gdrive://file/<id>`, поэтому файлы должны лежать в Google Drive.
**Загрузка больших файлов напрямую из скилла невозможна** — пользователь делает это вручную.

**2a — Найти папку `claude_for_reels`:**

```
mcp__Google_Drive__search_files:
  query: "title = 'claude_for_reels' and mimeType = 'application/vnd.google-apps.folder'"
```

Запиши `folder_id` из результата.

**2b — Посмотреть что уже загружено:**

```
mcp__Google_Drive__search_files:
  query: "parentId = '<folder_id>' and mimeType contains 'video/'"
```

Построй карту: `filename → gdrive_file_id` для всех найденных файлов.

**2c — Если нужных видео нет или папка пустая — СТОП, попроси пользователя:**

> 📂 **Нужно загрузить видео вручную**
>
> Видео в папке `claude_for_reels` на Google Drive не найдены (или нет нужных файлов).
>
> Пожалуйста, загрузи видео сам:
> 1. Открой [Google Drive](https://drive.google.com) → папку **claude_for_reels**
> 2. Перетащи туда нужные `.MOV` / `.mp4` файлы
> 3. Дождись окончания загрузки
> 4. Напиши мне «готово» — и я продолжу
>
> ⏳ Жду твоего сигнала.

**Не продолжай до тех пор, пока пользователь не напишет, что загрузка завершена.**

**2d — После подтверждения — повтори поиск:**

Снова выполни шаг 2b. После этого у тебя есть полная карта:
```
IMG_2778.MOV → gdrive_file_id: 1AbCd...
IMG_2910.MOV → gdrive_file_id: 1XyZw...
```

**→ TaskUpdate Step 2:** `completed`

---

## Step 3 — Analyze each video with CloudGlue

**→ TaskUpdate Step 3:** `in_progress`

CloudGlue-анализ каждого видео кешируется в файле `video_description.json`, который лежит в той же папке `claude_for_reels` на Google Drive (ключ — имя видеофайла). Это позволяет не гонять уже проанализированные видео через CloudGlue повторно.

**3a — Найти или создать `video_description.json`:**

```
Google Drive:search_files
  query: "parentId = '<folder_id>' and title = 'video_description.json'"
```

- **Если файл найден:** скачай и распарси его (`Google Drive:download_file_content`). Это карта `{ "IMG_2778.MOV": { ...описание CloudGlue... }, ... }`.
- **Если файла нет:** создай его пустым — `Google Drive:create_file` с содержимым `{}` в папке `claude_for_reels`, именем `video_description.json`, mime type `application/json`. Запомни его `file_id`.

**3b — Определить, какие видео нужно анализировать:**

Сравни список видео из Шага 2 (`filename → gdrive_file_id`) с ключами (именами файлов) уже сохранёнными в `video_description.json`.

- Видео, чьё имя **уже есть** в файле → пропускаем CloudGlue, берём готовое описание из JSON.
- Видео, чьего имени в файле **нет** → отправляем на анализ (только их).

Если все нужные видео уже есть в `video_description.json` — CloudGlue вообще не вызывается, сразу переходи к Шагу 4.

**3c — Анализ новых видео через CloudGlue:**

Для каждого видео, которого не было в `video_description.json`, вызови `Cloudglue:describe_video` используя CloudGlue file ID (из `list_videos`) — **не** gdrive URL:

```
url: "cloudglue://files/<cloudglue_file_id>"
config: {
  "enable_speech": true,
  "enable_visual_scene_description": true,
  "enable_scene_text": true,
  "enable_summary": false,
  "enable_audio_description": false
}
```

> **Important:** Always use `cloudglue://files/<id>`, not `gdrive://file/<id>`. The gdrive URL returns empty descriptions even when the video is indexed.

> **Timeout note:** Call videos **one at a time** — parallel calls time out. Retry once if timeout.

Для каждого нового видео зафиксируй:
- Что происходит визуально (пейзаж, люди, движение)
- **Точные таймкоды интересных моментов** — используются в Шаге 4

**3d — Сохранить результаты в `video_description.json`:**

Добавь описание каждого нового видео в JSON-карту по ключу — имени файла, и загрузи обновлённый файл обратно на Google Drive (`Google Drive:update_file` с тем же `file_id` из шага 3a, перезаписывая содержимое).

Итог: `video_description.json` содержит описания **всех** видео (и старых, и только что проанализированных) — именно эта объединённая карта используется дальше, в Шаге 4.

**→ TaskUpdate Step 3:** `completed`

---

## Step 4 — Select best segments based on CloudGlue timecodes

**→ TaskUpdate Step 4:** `in_progress`

Using **only** CloudGlue descriptions and timecodes, pick **2–4 segments** matching the theme. Combined duration **≤ 6 seconds**.

For each segment:
```
source_file: IMG_2778.MOV
start_time:  00:01.5
end_time:    00:03.5
duration:    2.0s
why:         Motion, ocean background, matches theme
```

**Hook** (shown on the Reel):
- Max 5–6 words, ALL CAPS
- Provocative, emotional, stop-scroll
- Examples: `"ТЫ ЕЩЁ НЕ БЫЛ ЗДЕСЬ?"` / `"МИР ЖДЁТ. А ТЫ?"` / `"ДОРОГА. ВЕТЕР. СВОБОДА."`

**Instagram caption**: 2–3 punchy sentences + 5–8 hashtags.

**→ TaskUpdate Step 4:** `completed`

---

## Step 5 — Cut and concat clips on device

**→ TaskUpdate Step 5:** `in_progress`

Run via `device_bash` (ffmpeg on user's Mac — no staging needed).

**Find the correct mount path first:**
```bash
echo $HOME && ls $HOME/mnt/
# PROJ = $HOME/mnt/<folder-name>
```

**Cut each segment — normalize to 30fps, 1080x1920, no audio:**
```bash
PROJ="$HOME/mnt/<folder-name>"

ffmpeg -ss <start1> -i "$PROJ/<file1>" -t <dur1> \
  -vf "scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,fps=30" \
  -c:v libx264 -preset ultrafast -crf 30 -an -r 30 /tmp/s1.mp4 -y 2>/dev/null

# (repeat for each segment as /tmp/s2.mp4, etc.)
```

> **Critical — always normalize fps.** Without `-vf fps=30 -r 30`, concat produces wrong durations.

**Verify durations:**
```bash
for f in /tmp/s1.mp4 /tmp/s2.mp4; do
  echo "$f: $(ffprobe -v quiet -show_entries format=duration -of csv=p=0 $f)s"
done
```

**Concat:**
```bash
printf "file '/tmp/s1.mp4'\nfile '/tmp/s2.mp4'\n" > /tmp/c.txt
ffmpeg -f concat -safe 0 -i /tmp/c.txt -c copy /tmp/clip_final.mp4 -y 2>/dev/null
echo "result: $(ffprobe -v quiet -show_entries format=duration -of csv=p=0 /tmp/clip_final.mp4)s"
cp /tmp/clip_final.mp4 "$PROJ/clip_final.mp4"
```

**Extract color palette** for hook styling:
```bash
ffmpeg -ss 1 -i /tmp/clip_final.mp4 -vframes 1 /tmp/frame.jpg -y 2>/dev/null
python3 -c "
from PIL import Image
img = Image.open('/tmp/frame.jpg').convert('RGB')
w, h = img.size
for label, pos in [('top',(w//2,h//8)),('mid',(w//2,h//2)),('bot',(w//2,h*3//4))]:
    r,g,b = img.getpixel(pos)
    print(f'{label}: #{r:02X}{g:02X}{b:02X}')
"
```

Choose:
- **primary**: `#FFFFFF` (white) for dark video; `#111111` for light
- **accent**: most vivid sampled color — for text stroke and underline

**→ TaskUpdate Step 5:** `completed`

---

## Step 6 — Add hook via Hyperframes and save Reel

**→ TaskUpdate Step 6:** `in_progress`

All work runs on device via `device_bash`.

Hyperframes compositions are **ordinary HTML** where:
- HTML + CSS control appearance
- `data-start` and `data-duration` on elements control timing in **frames** (at project fps)
- `data-fps` and `data-duration` on `<body>` define the project timeline

**6a — Determine output filename and create project folder:**

Choose a short theme slug from the hook text (e.g. `svoboda`, `more`, `vy`). This becomes the filename.

```bash
PROJ="$HOME/mnt/<folder-name>"
OUT="$PROJ/reel_<theme>.mp4"   # e.g. reel_more.mp4 — final destination
mkdir -p "$PROJ/hf_reel"
cp /tmp/clip_final.mp4 "$PROJ/hf_reel/clip.mp4"
```

**6b — Calculate frame counts** (30fps):

```
total_frames = clip_duration_seconds × 30
e.g. 6.0s × 30 = 180 frames
underline delay ≈ 0.4s × 30 = 12 frames
```

**6c — Write `$PROJ/hf_reel/composition.html`:**

Replace `ТЕКСТ ХУКА`, colors, and frame counts with actual values from Steps 4–5:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { width: 1080px; height: 1920px; overflow: hidden; background: #000; }

    .scene { position: relative; width: 100%; height: 100%; }

    video {
      position: absolute;
      width: 100%; height: 100%;
      object-fit: cover;
    }

    .hook {
      position: absolute;
      top: 11%;
      width: 100%;
      text-align: center;
      font-family: 'Arial Black', Arial, Helvetica, sans-serif;
      font-size: 96px;
      font-weight: 900;
      color: #FFFFFF;                    /* ← primary from Step 5 */
      -webkit-text-stroke: 4px #D4893A; /* ← accent from Step 5 */
      text-transform: uppercase;
      letter-spacing: 3px;
      line-height: 1.1;
      padding: 0 56px;
      animation: hookIn 0.5s ease forwards;
    }

    .underline {
      position: absolute;
      top: calc(11% + 120px);
      left: calc(50% - 100px);
      width: 200px;
      height: 5px;
      border-radius: 3px;
      background: #D4893A;              /* ← accent from Step 5 */
      transform-origin: left center;
      animation: wipeIn 0.4s ease forwards;
    }

    @keyframes hookIn {
      from { opacity: 0; transform: scale(0.85); }
      to   { opacity: 1; transform: scale(1); }
    }

    @keyframes wipeIn {
      from { transform: scaleX(0); }
      to   { transform: scaleX(1); }
    }
  </style>
</head>
<!-- data-duration in frames: clip_seconds × 30 -->
<body data-fps="30" data-duration="180">
  <div class="scene">

    <video
      src="clip.mp4"
      data-start="0"
      data-duration="180"
      muted playsinline>
    </video>

    <!-- Hook text: appears at frame 0 -->
    <div class="hook"
         data-start="0"
         data-duration="180">
      ТЕКСТ ХУКА
    </div>

    <!-- Underline: appears at frame 12 (≈0.4s) -->
    <div class="underline"
         data-start="12"
         data-duration="168">
    </div>

  </div>
</body>
</html>
```

**6d — Render directly into the project folder:**

```bash
cd "$PROJ/hf_reel"
npx hyperframes render --output "$OUT" 2>&1 | tail -15
```

> The `--output` flag takes the full absolute path, so the file lands directly in the project folder — no copy step needed.

**Verify:**
```bash
ffprobe -v quiet -show_entries format=duration -of csv=p=0 "$OUT"
echo "Saved: $OUT"
```

**If render fails — common fixes:**
- `data-duration` must be in frames, not seconds — recalculate
- Video `src` must be relative to the composition file (just `clip.mp4`, not absolute)
- Missing `data-fps` on `<body>` — add it
- CLI not found: try `npx hyperframes@latest render` instead
- Remove `| tail -15` temporarily to see the full error

**Не отправлять на телефон/веб.** Ролик уже лежит локально в `$OUT` (в папке проекта на Mac, рядом с остальными видео) — этого достаточно, дополнительная отправка не нужна и не выполняется.

**→ TaskUpdate Step 6:** `completed`

После сохранения выведи:
- ✅ Сохранено локально: `<path on Mac>` (папка с видео пользователя)
- 📝 Hook: `"ТЕКСТ ХУКА"`
- 📸 Instagram caption (ready to paste)