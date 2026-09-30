[Dcode.html](https://github.com/user-attachments/files/32872789/Dcode.html)
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>D code - محرر نصوص متعدد الملفات والمعاينة المباشرة</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- FontAwesome Icons -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <!-- Google Fonts -->
    <link href="https://fonts.googleapis.com/css2?family=Cairo:wght@300;400;600;700;800&family=JetBrains+Mono:ital,wght@0,400;0,500;0,700;1,400&display=swap" rel="stylesheet">
    <!-- JSZip Library لتحميل المشروع مضغوطاً -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
    <!-- TypeScript Compiler (Browser Support) -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/typescript/5.0.4/typescript.min.js"></script>
    <!-- CoffeeScript Compiler -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/coffeescript/2.7.0/coffeescript.min.js"></script>
    
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        cairo: ['Cairo', 'sans-serif'],
                        mono: ['JetBrains Mono', 'monospace'],
                    },
                    colors: {
                        matrix: {
                            green: '#00ff66',
                            darkgreen: '#003b15',
                            glow: 'rgba(0, 255, 102, 0.4)',
                            bg: '#050a06'
                        }
                    }
                }
            }
        }
    </script>

    <style>
        * { box-sizing: border-box; user-select: none; }
        body {
            font-family: 'Cairo', sans-serif;
            background-color: #030704;
            color: #e2e8f0;
            overflow: hidden;
            height: 100vh;
            margin: 0;
        }

        /* Matrix Background Canvas */
        #matrixCanvas {
            position: fixed;
            top: 0; left: 0; width: 100vw; height: 100vh;
            z-index: 0; pointer-events: none;
        }

        /* Glassmorphism styling */
        .glass-card {
            background: rgba(8, 18, 12, 0.85);
            backdrop-filter: blur(16px);
            -webkit-backdrop-filter: blur(16px);
            border: 1px solid rgba(0, 255, 102, 0.25);
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.8), 0 0 15px rgba(0, 255, 102, 0.1);
        }

        .editor-textarea {
            user-select: text !important;
            font-family: 'JetBrains Mono', monospace;
            caret-color: #00ff66;
            line-height: 1.6;
            tab-size: 4;
        }

        /* Custom Glow Scrollbar */
        ::-webkit-scrollbar { width: 6px; height: 6px; }
        ::-webkit-scrollbar-track { background: rgba(0, 20, 10, 0.5); }
        ::-webkit-scrollbar-thumb { background: rgba(0, 255, 102, 0.3); border-radius: 4px; }
        ::-webkit-scrollbar-thumb:hover { background: rgba(0, 255, 102, 0.7); }

        .matrix-btn {
            transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
        }
        .matrix-btn:hover {
            box-shadow: 0 0 15px rgba(0, 255, 102, 0.5);
            border-color: #00ff66;
        }

        .file-item { transition: all 0.2s; }
        .file-item.active {
            background: rgba(0, 255, 102, 0.15);
            border-right: 3px solid #00ff66;
            color: #00ff66;
        }

        .editor-textarea::-webkit-scrollbar { width: 8px; }

        #previewFrame {
            background-color: white;
            border-radius: 0 0 0.75rem 0.75rem;
        }
    </style>
</head>
<body class="relative flex p-2 md:p-4 gap-4 h-screen">

    <canvas id="matrixCanvas"></canvas>

    <!-- واجهة التطبيق الرئيسية -->
    <div class="relative z-10 w-full h-full flex flex-col md:flex-row gap-4">
        
        <!-- الشريط الجانبي (قائمة الملفات) -->
        <aside class="glass-card rounded-xl flex flex-col w-full md:w-64 flex-shrink-0 h-48 md:h-full border border-matrix-green/30">
            <!-- الهيدر -->
            <div class="p-4 border-b border-matrix-green/20 flex items-center justify-between">
                <div class="flex items-center gap-3">
                    <div class="w-8 h-8 rounded-lg bg-matrix-green/20 border border-matrix-green flex items-center justify-center text-matrix-green font-black text-lg">
                        D
                    </div>
                    <h1 class="font-black text-white tracking-wide text-lg">
                        D <span class="text-matrix-green">code</span>
                    </h1>
                </div>
            </div>

            <!-- قائمة الملفات -->
            <div class="flex-1 overflow-y-auto py-2" id="fileList">
                <!-- يتم حقن القائمة عبر الجافاسكربت -->
            </div>

            <!-- أزرار العمليات -->
            <div class="p-3 border-t border-matrix-green/20 flex flex-col gap-2">
                <button onclick="createNewFile()" class="matrix-btn bg-slate-900 hover:bg-slate-800 text-matrix-green border border-matrix-green/40 px-3 py-2 rounded-lg flex items-center justify-center gap-2 text-sm font-bold">
                    <i class="fa-solid fa-plus"></i> ملف جديد
                </button>
                <button onclick="downloadProject()" class="matrix-btn bg-matrix-green text-black px-3 py-2 rounded-lg flex items-center justify-center gap-2 text-sm font-bold shadow-[0_0_10px_rgba(0,255,102,0.4)]">
                    <i class="fa-solid fa-file-zipper"></i> تنزيل كملف واحد (ZIP)
                </button>
            </div>
        </aside>

        <!-- منطقة العمل الرئيسية -->
        <main class="flex-1 flex flex-col lg:flex-row gap-4 h-full min-h-0">
            
            <!-- نافذة محرر الأكواد -->
            <div class="glass-card rounded-xl flex-1 flex flex-col border border-matrix-green/30 min-h-0">
                <div class="bg-black/60 px-4 py-2 border-b border-matrix-green/20 flex justify-between items-center">
                    <div class="flex items-center gap-2 text-sm font-mono text-slate-300" id="currentFileHeader">
                        <!-- معلومات الملف الحالي -->
                    </div>
                    <button onclick="deleteCurrentFile()" class="text-rose-400 hover:text-rose-300 transition" title="حذف الملف الحالي">
                        <i class="fa-solid fa-trash-can"></i>
                    </button>
                </div>
                <div class="flex-1 relative bg-black/40">
                    <textarea 
                        id="codeEditor" 
                        dir="ltr" 
                        spellcheck="false"
                        oninput="handleInput()"
                        onkeydown="handleKeyDown(event)"
                        class="editor-textarea absolute inset-0 w-full h-full p-4 bg-transparent text-slate-100 placeholder-slate-600 focus:outline-none resize-none text-sm border-none"
                    ></textarea>
                </div>
            </div>

            <!-- نافذة المعاينة المباشرة -->
            <div id="previewContainer" class="glass-card rounded-xl flex-1 flex flex-col border border-matrix-green/30 min-h-0 relative">
                <div class="bg-black/60 px-4 py-2 border-b border-matrix-green/20 flex justify-between items-center">
                    <div class="flex items-center gap-2 text-sm font-bold text-slate-300">
                        <i class="fa-solid fa-eye text-matrix-green"></i>
                        <span>المعاينة المباشرة والربط والتنفيذ</span>
                    </div>
                    <button onclick="toggleFullScreen()" class="matrix-btn bg-slate-800 border border-slate-600 text-white px-2 py-1 rounded hover:text-matrix-green text-xs flex items-center gap-1">
                        <i class="fa-solid fa-expand"></i> ملء الشاشة
                    </button>
                </div>
                <!-- Iframe لعرض النتائج والمعاينة -->
                <iframe id="previewFrame" class="w-full flex-1 border-none bg-white"></iframe>
            </div>

        </main>
    </div>

    <script>
        // --- 1. خلفية تأثير Matrix ---
        const canvas = document.getElementById('matrixCanvas');
        const ctx = canvas.getContext('2d');
        function resizeCanvas() { canvas.width = window.innerWidth; canvas.height = window.innerHeight; }
        resizeCanvas();
        const alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@#$%&*+=';
        const fontSize = 16;
        let columns = Math.floor(canvas.width / fontSize);
        let drops = [];
        function initDrops() {
            columns = Math.floor(canvas.width / fontSize);
            drops = [];
            for (let i = 0; i < columns; i++) drops[i] = Math.random() * -100;
        }
        initDrops();
        window.addEventListener('resize', () => { resizeCanvas(); initDrops(); });
        function drawMatrix() {
            ctx.fillStyle = 'rgba(3, 7, 4, 0.08)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.font = `${fontSize}px JetBrains Mono, monospace`;
            for (let i = 0; i < drops.length; i++) {
                const text = alphabet.charAt(Math.floor(Math.random() * alphabet.length));
                ctx.fillStyle = Math.random() > 0.88 ? '#ffffff' : '#00ff66';
                const x = i * fontSize;
                const y = drops[i] * fontSize;
                ctx.fillText(text, x, y);
                if (y > canvas.height && Math.random() > 0.975) drops[i] = 0;
                drops[i]++;
            }
        }
        setInterval(drawMatrix, 33);

        // --- 2. إدارة ملفات مشروع D code ---
        let files = [
            { 
                id: 1, 
                name: 'index.html', 
                content: `<!DOCTYPE html>\n<html lang="ar" dir="rtl">\n<head>\n  <meta charset="UTF-8">\n  <link rel="stylesheet" href="style.css">\n</head>\n<body>\n  <div class="card">\n    <h1>مرحباً بك في D code!</h1>\n    <p id="output">جاري فتح وتجميع كافة الملفات المرتبطة...</p>\n  </div>\n\n  <!-- ربط ملفات المشروع المختلفة -->\n  <script src="script.js"><\/script>\n  <script src="app.ts"><\/script>\n  <script src="logic.coffee"><\/script>\n</body>\n</html>` 
            },
            {
                id: 2,
                name: 'style.css',
                content: `body {\n  font-family: system-ui, sans-serif;\n  background: #0f172a;\n  color: #fff;\n  display: flex;\n  justify-content: center;\n  align-items: center;\n  height: 100vh;\n  margin: 0;\n}\n.card {\n  background: rgba(255, 255, 255, 0.05);\n  padding: 30px;\n  border-radius: 16px;\n  border: 1px solid #00ff66;\n  box-shadow: 0 0 20px rgba(0,255,102,0.2);\n  text-align: center;\n}`
            },
            {
                id: 3,
                name: 'script.js',
                content: `// استدعاء ملف JSON محلي وقراءته تلقائياً\nfetch('data.json')\n  .then(res => res.json())\n  .then(data => {\n    console.log("JSON Loaded:", data);\n  });`
            },
            {
                id: 4,
                name: 'app.ts',
                content: `// كود TypeScript يتم تجميعه تلقائياً\ninterface User {\n  name: string;\n  status: string;\n}\n\nconst user: User = { name: "D code User", status: "Active" };\nconsole.log("TypeScript Executed:", user);`
            },
            {
                id: 5,
                name: 'logic.coffee',
                content: `# كود CoffeeScript يتم تجميعه تلقائياً\nsquare = (x) -> x * x\nconsole.log "CoffeeScript Executed! Square of 5 is:", square(5)`
            },
            {
                id: 6,
                name: 'data.json',
                content: `{\n  "appName": "D code",\n  "version": "3.0",\n  "status": "Ready"\n}`
            }
        ];

        let activeFileId = 1;
        const editor = document.getElementById('codeEditor');
        const previewFrame = document.getElementById('previewFrame');
        const fileListEl = document.getElementById('fileList');
        const currentFileHeaderEl = document.getElementById('currentFileHeader');

        // الحصول على أيقونة حسب نوع الملف
        function getFileIconInfo(fileName) {
            const ext = fileName.split('.').pop().toLowerCase();
            switch (ext) {
                case 'html': case 'htm':
                    return { icon: 'fa-brands fa-html5', color: 'text-orange-500' };
                case 'css':
                    return { icon: 'fa-brands fa-css3-alt', color: 'text-sky-400' };
                case 'js':
                    return { icon: 'fa-brands fa-js', color: 'text-yellow-400' };
                case 'ts':
                    return { icon: 'fa-solid fa-code', color: 'text-blue-400' };
                case 'coffee':
                    return { icon: 'fa-solid fa-mug-hot', color: 'text-amber-600' };
                case 'json':
                    return { icon: 'fa-solid fa-file-code', color: 'text-emerald-400' };
                default:
                    return { icon: 'fa-solid fa-file', color: 'text-slate-400' };
            }
        }

        function renderFileList() {
            fileListEl.innerHTML = '';
            files.forEach(file => {
                const isActive = file.id === activeFileId;
                const iconInfo = getFileIconInfo(file.name);
                const div = document.createElement('div');
                div.className = `file-item cursor-pointer px-4 py-2.5 text-sm font-mono flex items-center justify-between border-b border-matrix-green/10 text-slate-300 hover:bg-white/5 ${isActive ? 'active' : ''}`;
                div.onclick = () => switchFile(file.id);
                div.innerHTML = `
                    <div class="flex items-center gap-2 overflow-hidden">
                        <i class="${iconInfo.icon} ${iconInfo.color}"></i>
                        <span class="truncate">${file.name}</span>
                    </div>
                `;
                fileListEl.appendChild(div);
            });
        }

        function switchFile(id) {
            activeFileId = id;
            const file = files.find(f => f.id === id);
            editor.value = file.content;
            
            const iconInfo = getFileIconInfo(file.name);
            currentFileHeaderEl.innerHTML = `
                <i class="${iconInfo.icon} ${iconInfo.color}"></i>
                <span>${file.name}</span>
            `;

            renderFileList();
            updatePreview();
        }

        function handleInput() {
            const file = files.find(f => f.id === activeFileId);
            if(file) {
                file.content = editor.value;
                updatePreview();
            }
        }

        function createNewFile() {
            const newName = prompt("أدخل اسم الملف الجديد مع امتداده (مثل: main.ts, style.css, data.json):");
            if(!newName) return;

            if(files.some(f => f.name.toLowerCase() === newName.toLowerCase())) {
                alert("يوجد ملف بنفس هذا الاسم بالفعل!");
                return;
            }

            const ext = newName.split('.').pop().toLowerCase();
            let defaultContent = '';
            if(ext === 'html') defaultContent = '<!DOCTYPE html>\n<html>\n<head>\n</head>\n<body>\n\n</body>\n</html>';
            else if(ext === 'json') defaultContent = '{\n  \n}';

            const newFile = {
                id: Date.now(),
                name: newName,
                content: defaultContent
            };
            files.push(newFile);
            switchFile(newFile.id);
        }

        function deleteCurrentFile() {
            if(files.length <= 1) {
                alert("لا يمكن حذف الملف الأخير في المشروع!");
                return;
            }
            if(confirm("هل أنت متأكد من حذف هذا الملف؟")) {
                files = files.filter(f => f.id !== activeFileId);
                switchFile(files[0].id);
            }
        }

        // --- 3. نظام المعاينة والربط الذكي بين الملفات والتجميع التلقائي ---
        function updatePreview() {
            const activeFile = files.find(f => f.id === activeFileId);
            let htmlContent = '';

            // إذا كان الملف المحدد HTML يُعرض مباشرة مع ربطه ببقية الأصول
            if (activeFile && activeFile.name.endsWith('.html')) {
                htmlContent = activeFile.content;
            } else {
                // إذا لم يكن ملف HTML، ابحث عن ملف index.html في المشروع
                const mainHtml = files.find(f => f.name.toLowerCase() === 'index.html') || files.find(f => f.name.endsWith('.html'));
                if (mainHtml) {
                    htmlContent = mainHtml.content;
                } else {
                    // إغلاق العرض الخاص بملفات JSON أو CSS أو JS بشكل غير المباشر
                    if (activeFile.name.endsWith('.json')) {
                        let parsed = '';
                        try {
                            parsed = JSON.stringify(JSON.parse(activeFile.content), null, 2);
                            htmlContent = `<pre style="color:#00ff66; background:#111; padding:20px; font-family:monospace; margin:0; height:100vh; overflow:auto;">// JSON Valid:\n${parsed}</pre>`;
                        } catch(e) {
                            htmlContent = `<pre style="color:#ff4444; background:#111; padding:20px; font-family:monospace; margin:0; height:100vh;">// JSON Syntax Error:\n${e.message}</pre>`;
                        }
                    } else {
                        htmlContent = `<!DOCTYPE html><html><head><style>body{font-family:sans-serif; background:#111; color:#fff; padding:20px;}</style></head><body><h3>معاينة ملف: ${activeFile.name}</h3><p>للمعاينة الشاملة مع الربط، قم بالانتقال إلى ملف HTML المباشر.</p></body></html>`;
                    }
                }
            }

            // إنشاء بيئة المعاينة مع ربط الأصول واعتراض طلبات fetch والمكتبات المدمجة
            buildVirtualIframe(htmlContent);
        }

        function buildVirtualIframe(htmlRaw) {
            const parser = new DOMParser();
            const doc = parser.parseFromString(htmlRaw, 'text/html');

            // 1. معالجة واستبدال روابط CSS <link rel="stylesheet" href="...">
            const links = doc.querySelectorAll('link[rel="stylesheet"]');
            links.forEach(link => {
                const href = link.getAttribute('href');
                if (href) {
                    const targetFile = files.find(f => f.name === href || f.name === href.replace('./', ''));
                    if (targetFile) {
                        const style = doc.createElement('style');
                        style.textContent = targetFile.content;
                        link.parentNode.replaceChild(style, link);
                    }
                }
            });

            // 2. معالجة وسوم السكريبتات <script src="..."> لدعم JS, TS, CoffeeScript
            const scripts = doc.querySelectorAll('script[src]');
            scripts.forEach(script => {
                const src = script.getAttribute('src');
                if (src) {
                    const targetFile = files.find(f => f.name === src || f.name === src.replace('./', ''));
                    if (targetFile) {
                        const ext = targetFile.name.split('.').pop().toLowerCase();
                        let compiledCode = targetFile.content;

                        // تجميع TypeScript
                        if (ext === 'ts' && window.ts) {
                            try {
                                compiledCode = window.ts.transpile(targetFile.content, {
                                    target: window.ts.ScriptTarget.ES2015,
                                    module: window.ts.ModuleKind.None
                                });
                            } catch(err) {
                                compiledCode = `console.error("TypeScript Error in ${targetFile.name}:", ${JSON.stringify(err.message)});`;
                            }
                        } 
                        // تجميع CoffeeScript
                        else if (ext === 'coffee' && window.CoffeeScript) {
                            try {
                                compiledCode = window.CoffeeScript.compile(targetFile.content, { bare: true });
                            } catch(err) {
                                compiledCode = `console.error("CoffeeScript Error in ${targetFile.name}:", ${JSON.stringify(err.message)});`;
                            }
                        }

                        const newScript = doc.createElement('script');
                        newScript.textContent = compiledCode;
                        script.parentNode.replaceChild(newScript, script);
                    }
                }
            });

            // 3. حقن محاكي نظام الملفات الاصطناعي لاختطاف fetch() و XMLHttpRequest لدعم JSON و استدعاءات البيانات
            const virtualFSInjector = doc.createElement('script');
            const fileMapJSON = JSON.stringify(files.reduce((acc, f) => { acc[f.name] = f.content; return acc; }, {}));
            
            virtualFSInjector.textContent = `
                (function() {
                    const virtualFiles = ${fileMapJSON};
                    
                    // اختطاف fetch ليدعم قراءة الملفات المحلية المفتوحة في D code
                    const originalFetch = window.fetch;
                    window.fetch = function(url, options) {
                        const cleanUrl = url.replace(/^\.\//, '');
                        if (virtualFiles.hasOwnProperty(cleanUrl)) {
                            const content = virtualFiles[cleanUrl];
                            return Promise.resolve(new Response(content, {
                                status: 200,
                                headers: { 'Content-Type': cleanUrl.endsWith('.json') ? 'application/json' : 'text/plain' }
                            }));
                        }
                        return originalFetch.apply(this, arguments);
                    };
                })();
            `;

            if (doc.head) {
                doc.head.insertBefore(virtualFSInjector, doc.head.firstChild);
            }

            // كتابة النتيجة الشاملة داخل الـ iframe
            const iframeDoc = previewFrame.contentDocument || previewFrame.contentWindow.document;
            iframeDoc.open();
            iframeDoc.write(doc.documentElement.outerHTML);
            iframeDoc.close();
        }

        // التعامل مع زر Tab داخل المحرر
        function handleKeyDown(e) {
            if (e.key === 'Tab') {
                e.preventDefault();
                const start = editor.selectionStart;
                const end = editor.selectionEnd;
                editor.value = editor.value.substring(0, start) + "    " + editor.value.substring(end);
                editor.selectionStart = editor.selectionEnd = start + 4;
                handleInput();
            }
        }

        // --- 4. ملء الشاشة للمعاينة ---
        function toggleFullScreen() {
            const container = document.getElementById('previewContainer');
            if (!document.fullscreenElement) {
                if (container.requestFullscreen) container.requestFullscreen();
                else if (container.webkitRequestFullscreen) container.webkitRequestFullscreen();
            } else {
                if (document.exitFullscreen) document.exitFullscreen();
            }
        }

        // --- 5. تحميل المشروع كاملاً كملف ZIP واحد ---
        function downloadProject() {
            if(typeof JSZip === 'undefined') {
                alert("جاري تحميل مكتبة الضغط، يرجى إعادة المحاولة بعد ثوانٍ.");
                return;
            }
            const zip = new JSZip();
            files.forEach(file => {
                zip.file(file.name, file.content);
            });

            zip.generateAsync({type:"blob"}).then(function(content) {
                const link = document.createElement('a');
                link.href = URL.createObjectURL(content);
                link.download = "D_code_Project.zip";
                document.body.appendChild(link);
                link.click();
                document.body.removeChild(link);
            });
        }

        // التهيئة عند التحميل
        window.addEventListener('DOMContentLoaded', () => {
            switchFile(files[0].id);
        });
    </script>
</body>
</html>
