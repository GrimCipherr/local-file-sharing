import os
from pathlib import Path
from typing import List
from fastapi import FastAPI, File, UploadFile
from fastapi.responses import HTMLResponse, RedirectResponse, FileResponse
import aiofiles
import uvicorn
import datetime

UPLOAD_DIR = "shared"
os.makedirs(UPLOAD_DIR, exist_ok=True)
BASE_DIR = Path(UPLOAD_DIR).resolve()

app = FastAPI(title="Local File Share")

def safe_path(filename: str) -> Path:
    file_path = (BASE_DIR / filename).resolve()
    if not str(file_path).startswith(str(BASE_DIR)):
        raise ValueError("Invalid path")
    return file_path

def get_file_type(entry):
    if entry.is_dir():
        return "directory"
    ext = entry.name.split(".")[-1].lower() if "." in entry.name else ""
    image_ext = ["png", "jpg", "jpeg", "gif", "bmp", "svg"]
    video_ext = ["mp4", "webm", "avi", "mov", "mkv"]
    audio_ext = ["mp3", "wav", "ogg", "flac"]
    doc_ext = ["pdf", "doc", "docx", "txt", "rtf"]
    sheet_ext = ["xls", "xlsx", "csv"]
    pres_ext = ["ppt", "pptx"]
    archive_ext = ["zip", "rar", "tar", "gz"]
    exe_ext = ["exe", "apk", "bat"]
    if ext in image_ext:
        return "image"
    elif ext in video_ext:
        return "video"
    elif ext in audio_ext:
        return "audio"
    elif ext in doc_ext:
        return "document"
    elif ext in sheet_ext:
        return "sheet"
    elif ext in pres_ext:
        return "presentation"
    elif ext in archive_ext:
        return "archive"
    elif ext in exe_ext:
        return "executable"
    else:
        return "other"

def get_file_list():
    items = []
    with os.scandir(UPLOAD_DIR) as it:
        for entry in it:
            if entry.name.startswith("."):
                continue  # Skip hidden files
            info = entry.stat()
            file_type = get_file_type(entry)
            ext = entry.name.split(".")[-1].lower() if "." in entry.name else ""
            items.append({
                "name": entry.name,
                "type": file_type,
                "ext": ext,
                "size": info.st_size,
                "modified": datetime.datetime.fromtimestamp(info.st_mtime).strftime("%Y-%m-%d %H:%M:%S"),
                "modified_ts": info.st_mtime
            })
    items.sort(key=lambda x: x["name"].lower())
    return items

@app.get("/", response_class=HTMLResponse)
async def home():
    files = get_file_list()
    files_html = ""
    for f in files:
        thumbnail = ""
        if f["type"] == "directory":
            thumbnail = '<span class="icon">📁</span>'
            files_html += f"""
            <div class="file-item dir" data-name="{f['name'].lower()}" data-type="{f['type']}" data-size="{f['size']}" data-modified="{f['modified_ts']}">
                {thumbnail}
                <div class="file-info">
                    <span class="file-name">{f['name']}/</span>
                    <span class="file-meta">Directory | Modified: {f['modified']}</span>
                </div>
            </div>"""
        else:
            preview_class = "previewable" if f["type"] in ["image", "video", "audio", "document"] else ""
            if f["type"] == "image":
                thumbnail = f'<img src="/download/{f["name"]}" class="thumb" alt="thumb">'
            elif f["type"] == "video":
                thumbnail = f'<video src="/download/{f["name"]}" class="thumb" preload="metadata" muted></video>'
            elif f["type"] == "audio":
                thumbnail = '<span class="icon">🎵</span>'
            elif f["type"] == "document":
                thumbnail = '<span class="icon">📄</span>'
            elif f["type"] == "sheet":
                thumbnail = '<span class="icon">📊</span>'
            elif f["type"] == "presentation":
                thumbnail = '<span class="icon">📑</span>'
            elif f["type"] == "archive":
                thumbnail = '<span class="icon">🗜️</span>'
            elif f["type"] == "executable":
                thumbnail = '<span class="icon">⚙️</span>'
            else:
                thumbnail = '<span class="icon">📎</span>'
            files_html += f"""
            <div class="file-item {preview_class}" data-filename="{f['name']}" data-name="{f['name'].lower()}" data-type="{f['type']}" data-size="{f['size']}" data-modified="{f['modified_ts']}">
                <div class="thumbnail">{thumbnail}</div>
                <div class="file-info">
                    <span class="file-name">{f['name']}</span>
                    <span class="file-meta">Size: {f['size']//1024} KB | Modified: {f['modified']}</span>
                </div>
                <div class="file-actions">
                    <a href="/download/{f['name']}" class="btn">Download</a>
                    <a href="/delete/{f['name']}" class="btn delete">Delete</a>
                </div>
            </div>"""

    return f"""
    <html>
    <head>
        <title>Local File Share</title>
        <style>
            body {{
                margin: 0;
                font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
                background-color: #121212;
                color: #f0f0f0;
                display: flex;
                height: 100vh;
                overflow: hidden;
            }}
            #file-list {{
                width: 400px;
                border-right: 1px solid #333;
                padding: 20px;
                overflow-y: auto;
                background-color: #1e1e1e;
                box-shadow: 2px 0 5px rgba(0,0,0,0.5);
            }}
            #main {{
                flex-grow: 1;
                padding: 20px;
                overflow-y: auto;
                position: relative;
                transition: border 0.2s, background-color 0.2s;
            }}
            #main.drag-over {{
                border: 2px dashed #4FC3F7;
                background-color: rgba(79,195,247,0.1);
            }}
            h2 {{
                margin-top: 0;
                font-size: 1.5em;
                color: #ffffff;
            }}
            .file-item {{
                display: flex;
                align-items: center;
                margin-bottom: 10px;
                padding: 10px;
                background-color: #242424;
                border-radius: 8px;
                transition: background-color 0.2s;
            }}
            .file-item:hover {{
                background-color: #2c2c2c;
            }}
            .file-item.dir {{
                background-color: #282828;
            }}
            .thumbnail {{
                margin-right: 10px;
                width: 40px;
                height: 40px;
                display: flex;
                align-items: center;
                justify-content: center;
            }}
            .thumb {{
                width: 40px;
                height: 40px;
                object-fit: cover;
                border-radius: 4px;
                pointer-events: none;
            }}
            .icon {{
                font-size: 24px;
            }}
            .file-info {{
                flex-grow: 1;
                display: flex;
                flex-direction: column;
            }}
            .file-name {{
                font-weight: 600;
                color: #ffffff;
            }}
            .file-meta {{
                font-size: 0.85em;
                color: #aaaaaa;
            }}
            .file-actions {{
                display: flex;
            }}
            .file-actions a {{
                text-decoration: none;
                margin-left: 8px;
                padding: 6px 12px;
                border-radius: 6px;
                font-size: 0.85em;
                transition: background-color 0.2s;
            }}
            .file-actions a.btn {{
                background-color: #4FC3F7;
                color: #121212;
            }}
            .file-actions a.delete {{
                background-color: #f44336;
                color: #ffffff;
            }}
            .file-actions a:hover {{
                opacity: 0.9;
            }}
            input[type="submit"] {{
                margin-top: 10px;
                padding: 8px 16px;
                background-color: #4FC3F7;
                border: none;
                color: #121212;
                cursor: pointer;
                border-radius: 6px;
                font-weight: 600;
                transition: background-color 0.2s;
            }}
            input[type="submit"]:hover {{
                background-color: #039BE5;
            }}
            #progress-container {{
                margin-top: 10px;
                display: none;
                color: #aaaaaa;
            }}
            #preview {{
                margin-top: 20px;
                max-width: 100%;
                max-height: 500px;
                border-radius: 8px;
                overflow: hidden;
            }}
            #search {{
                width: 100%;
                padding: 8px;
                margin-bottom: 10px;
                border-radius: 6px;
                border: 1px solid #333;
                background-color: #242424;
                color: #f0f0f0;
            }}
            #sort-controls {{
                display: flex;
                align-items: center;
                margin-bottom: 15px;
            }}
            #sort-controls select {{
                flex-grow: 1;
                padding: 6px;
                border-radius: 6px;
                background-color: #242424;
                color: #f0f0f0;
                border: 1px solid #333;
                margin-right: 8px;
            }}
            #sort-controls button {{
                padding: 6px 12px;
                background-color: #333;
                color: #f0f0f0;
                border: none;
                border-radius: 6px;
                cursor: pointer;
                transition: background-color 0.2s;
            }}
            #sort-controls button:hover {{
                background-color: #444;
            }}
            #sort-controls button.active {{
                background-color: #4FC3F7;
                color: #121212;
            }}
            .file-progress {{
                display: flex;
                align-items: center;
                margin-bottom: 5px;
                padding: 5px;
                background-color: #242424;
                border-radius: 4px;
            }}
            .file-progress span {{
                flex: 1;
                margin-right: 10px;
                overflow: hidden;
                text-overflow: ellipsis;
                white-space: nowrap;
            }}
            .file-progress progress {{
                width: 150px;
                height: 8px;
            }}
            .file-progress button {{
                margin-left: 10px;
                padding: 4px 8px;
                background-color: #f44336;
                color: #ffffff;
                border: none;
                border-radius: 4px;
                cursor: pointer;
            }}
            .file-progress.cancelled {{
                color: #f44336;
            }}
            .file-progress.cancelled progress {{
                accent-color: #f44336;
            }}
            .summary {{
                cursor: pointer;
                font-weight: bold;
                display: flex;
                align-items: center;
                padding: 5px;
                background-color: #242424;
                border-radius: 4px;
            }}
            .circle-progress {{
                position: relative;
                width: 40px;
                height: 40px;
                cursor: pointer;
            }}
            .circle-progress svg {{
                transform: rotate(-90deg);
            }}
            .circle-progress circle {{
                fill: none;
                stroke-width: 4;
                stroke: #333;
            }}
            .circle-progress .progress {{
                stroke: #4FC3F7;
                transition: stroke-dashoffset 0.35s;
            }}
            .circle-progress span {{
                position: absolute;
                top: 50%;
                left: 50%;
                transform: translate(-50%, -50%);
                font-size: 12px;
                color: #ffffff;
            }}
            #modal {{
                display: none;
                position: fixed;
                top: 0;
                left: 0;
                width: 100%;
                height: 100%;
                background-color: rgba(0,0,0,0.5);
                justify-content: center;
                align-items: center;
                z-index: 1000;
            }}
            #modal-content {{
                background-color: #1e1e1e;
                padding: 20px;
                border-radius: 8px;
                width: 400px;
                max-height: 80vh;
                overflow-y: auto;
                position: relative;
            }}
            #modal-close {{
                position: absolute;
                top: 10px;
                right: 10px;
                cursor: pointer;
                color: #aaa;
                font-size: 20px;
            }}
            #modal-close:hover {{
                color: #fff;
            }}
            .details {{
                margin-top: 10px;
            }}
            .clickable {{
                cursor: pointer;
            }}
        </style>
    </head>
    <body>
        <div id="file-list">
            <h2>Files</h2>
            <input id="search" type="text" placeholder="Search files..." oninput="filterFiles()">
            <div id="sort-controls">
                <select id="sort-by">
                    <option value="name">Name</option>
                    <option value="size">Size</option>
                    <option value="modified">Date</option>
                    <option value="type">Type</option>
                </select>
                <button id="sort-asc" class="active" onclick="sortFiles(true)">↑</button>
                <button id="sort-desc" onclick="sortFiles(false)">↓</button>
            </div>
            <div id="files-container">
                {files_html}
            </div>
        </div>
        <div id="main">
            <h2>Upload File(s)</h2>
            <form id="upload-form" enctype="multipart/form-data" method="post">
                <input name="files" type="file" multiple>
                <input type="submit" value="Upload">
            </form>
            <div id="progress-container">
            </div>
            <div id="preview"></div>
        </div>
        <div id="modal">
            <div id="modal-content">
                <span id="modal-close">&times;</span>
                <h3>Uploading Files</h3>
                <div id="modal-details"></div>
            </div>
        </div>

        <script>
            const form = document.getElementById('upload-form');
            const progressContainer = document.getElementById('progress-container');
            const filesContainer = document.getElementById('files-container');
            const previewDiv = document.getElementById('preview');
            const searchInput = document.getElementById('search');
            const sortBySelect = document.getElementById('sort-by');
            const sortAscBtn = document.getElementById('sort-asc');
            const sortDescBtn = document.getElementById('sort-desc');
            const main = document.getElementById('main');
            const modal = document.getElementById('modal');
            const modalDetails = document.getElementById('modal-details');
            const modalClose = document.getElementById('modal-close');

            let currentSortAsc = true;
            let xhrs = [];  // To store xhr for cancellation

            // Drag-and-drop handlers
            main.addEventListener('dragenter', (e) => {{
                e.preventDefault();
                main.classList.add('drag-over');
            }});
            main.addEventListener('dragleave', (e) => {{
                e.preventDefault();
                main.classList.remove('drag-over');
            }});
            main.addEventListener('dragover', (e) => {{
                e.preventDefault();
            }});
            main.addEventListener('drop', (e) => {{
                e.preventDefault();
                main.classList.remove('drag-over');
                const files = e.dataTransfer.files;
                if (files.length) {{
                    uploadFiles(files);
                }}
            }});

            form.addEventListener('submit', async (e) => {{
                e.preventDefault();
                const files = form.files.files;
                if (!files.length) return;
                uploadFiles(files);
            }});

            async function uploadFiles(fileList) {{
                progressContainer.style.display = 'block';
                progressContainer.innerHTML = '';
                xhrs = [];  // Reset xhrs

                const filesArray = Array.from(fileList);
                let totalSize = filesArray.reduce((sum, file) => sum + file.size, 0);
                let fileProgress = filesArray.map((file, i) => {{
                    const progDiv = document.createElement('div');
                    progDiv.className = 'file-progress';
                    progDiv.innerHTML = `<span>${{file.name}}</span> <progress value="0" max="1"></progress> <button onclick="cancelUpload(${{i}})">Cancel</button>`;
                    return {{ div: progDiv, progress: progDiv.querySelector('progress'), loaded: 0, total: file.size, cancelled: false }};
                }});

                let summary = null;
                let overallPercentSpan = null;
                let circleProgress = null;

                if (filesArray.length > 1) {{
                    summary = document.createElement('div');
                    summary.className = 'circle-progress clickable';
                    summary.innerHTML = `
                        <svg width="40" height="40">
                            <circle cx="20" cy="20" r="18" />
                            <circle class="progress" cx="20" cy="20" r="18" stroke-dasharray="113" stroke-dashoffset="113" />
                        </svg>
                        <span id="overall-percent">0%</span>
                    `;
                    progressContainer.appendChild(summary);
                    circleProgress = summary.querySelector('.progress');
                    overallPercentSpan = document.getElementById('overall-percent');

                    summary.addEventListener('click', () => {{
                        modal.style.display = 'flex';
                        modalDetails.innerHTML = '';
                        fileProgress.forEach(p => modalDetails.appendChild(p.div.cloneNode(true)));
                    }});
                }} else {{
                    fileProgress.forEach(p => progressContainer.appendChild(p.div));
                }}

                modalClose.addEventListener('click', () => {{
                    modal.style.display = 'none';
                }});
                modal.addEventListener('click', (e) => {{
                    if (e.target === modal) {{
                        modal.style.display = 'none';
                    }}
                }});

                const promises = filesArray.map((file, i) => {{
                    return new Promise((resolve, reject) => {{
                        const formData = new FormData();
                        formData.append('files', file);

                        const xhr = new XMLHttpRequest();
                        xhrs[i] = xhr;
                        xhr.open('POST', '/upload/');

                        xhr.upload.onprogress = (e) => {{
                            if (fileProgress[i].cancelled) return;
                            if (e.lengthComputable) {{
                                fileProgress[i].loaded = e.loaded;
                                fileProgress[i].progress.value = e.loaded / e.total;
                                if (circleProgress) {{
                                    const totalLoaded = fileProgress.reduce((sum, p) => sum + (p.cancelled ? 0 : p.loaded), 0);
                                    const adjustedTotalSize = fileProgress.reduce((sum, p) => sum + (p.cancelled ? 0 : p.total), 0);
                                    const overallProgress = adjustedTotalSize > 0 ? totalLoaded / adjustedTotalSize : 0;
                                    const dashoffset = 113 * (1 - overallProgress);
                                    circleProgress.style.strokeDashoffset = dashoffset;
                                    overallPercentSpan.textContent = `${{Math.round(overallProgress * 100)}}%`;
                                }}
                            }}
                        }};

                        xhr.onload = () => {{
                            if (fileProgress[i].cancelled) {{
                                resolve();  // Still resolve to continue others
                                return;
                            }}
                            fileProgress[i].progress.value = 1;
                            resolve();
                        }};

                        xhr.onabort = () => {{
                            fileProgress[i].cancelled = true;
                            fileProgress[i].div.classList.add('cancelled');
                            fileProgress[i].div.querySelector('span').textContent += ' (Cancelled)';
                            fileProgress[i].div.querySelector('button').disabled = true;
                            resolve();  // Resolve to not block others
                        }};

                        xhr.onerror = () => {{
                            reject();
                        }};

                        xhr.send(formData);
                    }});
                }});

                await Promise.allSettled(promises);  // Allow cancellations to settle
                progressContainer.style.display = 'none';
                modal.style.display = 'none';
                form.reset();
                loadFiles();
            }}

            function cancelUpload(index) {{
                if (xhrs[index]) {{
                    xhrs[index].abort();
                }}
            }}

            async function loadFiles() {{
                const res = await fetch('/');
                const html = await res.text();
                const parser = new DOMParser();
                const doc = parser.parseFromString(html, 'text/html');
                const newFiles = doc.getElementById('files-container').innerHTML;
                filesContainer.innerHTML = newFiles;
                attachPreviewListeners();
                sortFiles(currentSortAsc);  // Re-sort after load
                filterFiles();  // Re-filter if search active
            }}

            function attachPreviewListeners() {{
                const previewables = document.querySelectorAll('.previewable');
                previewables.forEach(el => {{
                    el.onclick = () => {{
                        const name = el.dataset.filename;
                        const ext = name.split('.').pop().toLowerCase();
                        if(['png','jpg','jpeg','gif','bmp','svg'].includes(ext)) {{
                            previewDiv.innerHTML = `<img src="/download/${{name}}" alt="${{name}}" style="max-width:100%; max-height:500px;">`;
                        }} else if(['mp4','webm','avi','mov','mkv'].includes(ext)) {{
                            previewDiv.innerHTML = `<video controls style="max-width:100%; max-height:500px;"><source src="/download/${{name}}"></video>`;
                        }} else if(['mp3','wav','ogg','flac'].includes(ext)) {{
                            previewDiv.innerHTML = `<audio controls style="max-width:100%;"><source src="/download/${{name}}"></audio>`;
                        }} else if(['pdf','txt'].includes(ext)) {{
                            previewDiv.innerHTML = `<iframe src="/download/${{name}}" style="width:100%; height:500px;"></iframe>`;
                        }}
                    }}
                }});
            }}

            function filterFiles() {{
                const filter = searchInput.value.toLowerCase();
                document.querySelectorAll('#files-container .file-item').forEach(item => {{
                    const name = item.querySelector('.file-name')?.innerText.toLowerCase() || '';
                    item.style.display = name.includes(filter) ? 'flex' : 'none';
                }});
            }}

            function sortFiles(asc) {{
                currentSortAsc = asc;
                sortAscBtn.classList.toggle('active', asc);
                sortDescBtn.classList.toggle('active', !asc);

                const field = sortBySelect.value;
                const items = Array.from(filesContainer.querySelectorAll('.file-item'));
                items.sort((a, b) => {{
                    let va = a.dataset[field];
                    let vb = b.dataset[field];
                    if (field === 'size' || field === 'modified') {{
                        va = parseFloat(va);
                        vb = parseFloat(vb);
                        return asc ? va - vb : vb - va;
                    }} else {{
                        return asc ? va.localeCompare(vb) : vb.localeCompare(va);
                    }}
                }});
                items.forEach(item => filesContainer.appendChild(item));
                filterFiles();  // Re-apply filter after sort
            }}

            // Event listeners for sort
            sortBySelect.addEventListener('change', () => sortFiles(currentSortAsc));
            sortAscBtn.addEventListener('click', () => sortFiles(true));
            sortDescBtn.addEventListener('click', () => sortFiles(false));

            // Initial sort
            sortFiles(true);

            // Auto-refresh every 5 seconds
            setInterval(loadFiles, 5000);
            attachPreviewListeners();
        </script>
    </body>
    </html>
    """

@app.post("/upload/")
async def upload_files(files: List[UploadFile] = File(...)):
    for file in files:
        file_path = safe_path(file.filename)
        async with aiofiles.open(file_path, "wb") as f:
            while chunk := await file.read(1024*1024):
                await f.write(chunk)
    return RedirectResponse("/", status_code=303)

@app.get("/download/{filename}")
async def download_file(filename: str):
    try:
        file_path = safe_path(filename)
        if file_path.exists():
            return FileResponse(file_path, filename=filename)
        return {"error": "File not found"}
    except ValueError:
        return {"error": "Invalid filename"}

@app.get("/delete/{filename}")
async def delete_file(filename: str):
    try:
        file_path = safe_path(filename)
        if file_path.exists():
            os.remove(file_path)
            return RedirectResponse("/", status_code=303)
        return {"error": "File not found"}
    except ValueError:
        return {"error": "Invalid filename"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
