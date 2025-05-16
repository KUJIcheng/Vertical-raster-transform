<script>
    import { onMount } from 'svelte';

    let screenWidth = 0;
    let screenHeight = 0;
    let uploadedImage = null;
    let processedImage = null;

    // 当前选择的模式：raster = 光栅玻璃, rainbow = 虹玻璃
    let mode = "raster";

    // 每种模式的默认参数
    const defaultParams = {
        raster: {
            sliceCount: 20,
            blurRadius: 10,
            highlightIntensity: 0.1,
            shadowIntensity: 0.1
        },
        rainbow: {
            sliceCount: 20,
            blurRadius: 10,
            highlightIntensity: 0.1,
            shadowIntensity: 0.1,
            narrowRatioDivisor: 4
        }
    };

    // 初始化参数为 raster 模式
    let sliceCount = defaultParams.raster.sliceCount;
    let blurRadius = defaultParams.raster.blurRadius;
    let highlightIntensity = defaultParams.raster.highlightIntensity;
    let shadowIntensity = defaultParams.raster.shadowIntensity;
    let narrowRatioDivisor = defaultParams.rainbow.narrowRatioDivisor; // 仅 rainbow 用

    function handleModeChange(event) {
        mode = event.target.value;
        const params = defaultParams[mode];
        sliceCount = params.sliceCount;
        blurRadius = params.blurRadius;
        highlightIntensity = params.highlightIntensity;
        shadowIntensity = params.shadowIntensity;
        if (mode === "rainbow") {
            narrowRatioDivisor = params.narrowRatioDivisor;
        }
    }
    
    let pyodide = null;
    let pyodideReady = false;

    let isLoading = false;
    let isDownloadable = false;
    const defaultUploadedImage = 'icons8-photo-100.png';
    const defaultProcessedImage = 'icons8-animated-100.png';

    let originalFileName = ""

    let isEnglish = false;

    uploadedImage = defaultUploadedImage;
    processedImage = defaultProcessedImage;

    // 动态加载脚本函数
    const loadScript = async (url) => {
        return new Promise((resolve, reject) => {
            const script = document.createElement('script');
            script.src = url;
            script.async = true;
            script.onload = resolve;
            script.onerror = () => reject(new Error(`Failed to load script: ${url}`));
            document.head.appendChild(script);
        });
    };

    const updateDimensions = () => {
        screenWidth = window.innerWidth;
        screenHeight = window.innerHeight;
    };

    // 备份加载方程
    // const loadPyodideAndPackages = async () => {
    //     console.log("Loading Pyodide...");
    //     pyodide = await loadPyodide();
    //     await pyodide.loadPackage("Pillow");
    //     pyodideReady = true;
    //     console.log("Pyodide and Pillow loaded successfully.");
    // };

    let baseURL;

    if (typeof window !== "undefined") {
        const { protocol, host, pathname } = window.location;

        // 判断是否在 GitHub Pages
        const isGithubPages = pathname.startsWith("/Vertical-raster-transform");

        baseURL = `${protocol}//${host}${isGithubPages ? "/Vertical-raster-transform" : ""}`;
    }


    const loadPyodideAndPackages = async () => {
        console.log("Loading Pyodide...");

        pyodide = await loadPyodide({
            indexURL: `${baseURL}`
        });

        await pyodide.loadPackage("micropip");

        await pyodide.runPythonAsync(`
            import micropip
            await micropip.install("${baseURL}/numpy-2.0.2-cp312-cp312-pyodide_2024_0_wasm32.whl")
            await micropip.install("${baseURL}/pillow-10.2.0-cp312-cp312-pyodide_2024_0_wasm32.whl")
        `);

        pyodideReady = true;
        console.log("Pyodide and packages loaded successfully.");
    };

    const handleImageUpload = (event) => {
        const file = event.target.files[0];
        if (file) {
            originalFileName = file.name;
            const reader = new FileReader();
            reader.onload = (e) => {
                uploadedImage = e.target.result;
                isDownloadable = false;
            };
            reader.readAsDataURL(file);
        }
    };

    const handleConversion = async () => {
        if (!pyodideReady) {
            alert("Pyodide is still loading. Please wait...");
            return;
        }

        if (uploadedImage === defaultUploadedImage) {
            alert("Please upload an image first.");
            return;
        }

        isLoading = true;

        const imageData = uploadedImage.split(",")[1];

        let pythonCode = "";

        if (mode === "raster") {
            pythonCode = `
        from PIL import Image, ImageFilter, ImageDraw
        from io import BytesIO
        import base64

        def raster_blur_effect(data, file_type, slice_count, blur_radius, highlight_intensity, shadow_intensity):
            input_image = Image.open(BytesIO(base64.b64decode(data)))
            width, height = input_image.size

            num_slices = (slice_count + 1) // 2
            original_slice_width = int(width / num_slices)
            step_width = int(width / (num_slices / 0.5))
            compressed_slice_width = int(width / slice_count)

            slices = []
            for i in range(slice_count):
                start_x = int(i * step_width)
                end_x = min(start_x + original_slice_width, width)
                slice_strip = input_image.crop((start_x, 0, end_x, height))

                if blur_radius > 0:
                    slice_strip = slice_strip.filter(ImageFilter.GaussianBlur(blur_radius))

                if highlight_intensity > 0:
                    highlight_width = int(original_slice_width * 0.2)
                    gradient = Image.new('RGBA', (highlight_width, height), (255, 255, 255, 0))
                    gradient_draw = ImageDraw.Draw(gradient)
                    for x in range(highlight_width):
                        alpha = int(255 * highlight_intensity * (1 - x / highlight_width))
                        gradient_draw.line([(x, 0), (x, height)], fill=(255, 255, 255, alpha))
                    slice_strip.paste(gradient, (0, 0), gradient)

                if shadow_intensity > 0:
                    shadow_width = int(original_slice_width * 0.1)
                    gradient = Image.new('RGBA', (shadow_width, height), (0, 0, 0, 0))
                    gradient_draw = ImageDraw.Draw(gradient)
                    for x in range(shadow_width):
                        alpha = int(255 * shadow_intensity * (x / shadow_width))
                        gradient_draw.line([(x, 0), (x, height)], fill=(0, 0, 0, alpha))
                    slice_strip.paste(gradient, (original_slice_width - shadow_width, 0), gradient)

                slices.append(slice_strip.resize((compressed_slice_width, height), Image.Resampling.LANCZOS))

            final_image_width = compressed_slice_width * slice_count
            final_image = Image.new('RGB', (final_image_width, height))
            x_offset = 0
            for strip in slices:
                final_image.paste(strip, (x_offset, 0))
                x_offset += compressed_slice_width

            final_image = final_image.resize((width, height), Image.Resampling.LANCZOS)
            output_buffer = BytesIO()
            final_image.save(output_buffer, format="PNG")
            return base64.b64encode(output_buffer.getvalue()).decode("utf-8")

        result = raster_blur_effect("${imageData}", "PNG", ${sliceCount}, ${blurRadius}, ${highlightIntensity}, ${shadowIntensity})
        result
        `;
        } else if (mode === "rainbow") {
            pythonCode = `
        from PIL import Image, ImageFilter, ImageDraw
        import base64
        from io import BytesIO

        def raster_blur_effect3(data_base64, slice_count, blur_radius, highlight_intensity, shadow_intensity, narrow_ratio_divisor):
            input_image = Image.open(BytesIO(base64.b64decode(data_base64)))
            width, height = input_image.size
            total_leaves = 2 * slice_count - 1
            raw_leaf_width = width / total_leaves

            base_divisor = 6
            base_multiplier = 8
            scale_factor = 0.2
            narrow_sampling_multiplier = base_multiplier + scale_factor * max(0, (narrow_ratio_divisor - base_divisor))

            strips = []

            for i in range(total_leaves):
                is_wide = (i % 2 == 0)
                if is_wide:
                    start_x = int(i * raw_leaf_width)
                    end_x = int(start_x + raw_leaf_width)
                else:
                    center_x = int(i * raw_leaf_width + raw_leaf_width / 2)
                    half_width = int(raw_leaf_width * narrow_sampling_multiplier / 2)
                    start_x = max(0, center_x - half_width)
                    end_x = min(width, center_x + half_width)

                raw_strip = input_image.crop((start_x, 0, end_x, height))
                applied_blur = blur_radius
                if applied_blur > 0:
                    raw_strip = raw_strip.filter(ImageFilter.GaussianBlur(applied_blur))

                if highlight_intensity > 0:
                    highlight_w = int(raw_strip.width * 0.2)
                    gradient = Image.new('RGBA', (highlight_w, height), (255, 255, 255, 0))
                    draw = ImageDraw.Draw(gradient)
                    for x in range(highlight_w):
                        alpha = int(255 * highlight_intensity * (1 - x / highlight_w))
                        draw.line([(x, 0), (x, height)], fill=(255, 255, 255, alpha))
                    raw_strip.paste(gradient, (0, 0), gradient)

                if shadow_intensity > 0:
                    shadow_w = int(raw_strip.width * 0.1)
                    gradient = Image.new('RGBA', (shadow_w, height), (0, 0, 0, 0))
                    draw = ImageDraw.Draw(gradient)
                    for x in range(shadow_w):
                        alpha = int(255 * shadow_intensity * (x / shadow_w))
                        draw.line([(x, 0), (x, height)], fill=(0, 0, 0, alpha))
                    raw_strip.paste(gradient, (raw_strip.width - shadow_w, 0), gradient)

                compressed_width = int(raw_strip.width) if is_wide else int(raw_leaf_width / narrow_ratio_divisor)
                strip = raw_strip.resize((compressed_width, height), Image.Resampling.LANCZOS)
                strips.append(strip)

            combined_width = sum(s.width for s in strips)
            combined_image = Image.new('RGB', (combined_width, height))
            offset = 0
            for s in strips:
                combined_image.paste(s, (offset, 0))
                offset += s.width

            final_image = combined_image.resize((width, height), Image.Resampling.LANCZOS)
            output_buffer = BytesIO()
            final_image.save(output_buffer, format="PNG")
            return base64.b64encode(output_buffer.getvalue()).decode("utf-8")

        result = raster_blur_effect3("${imageData}", ${sliceCount}, ${blurRadius}, ${highlightIntensity}, ${shadowIntensity}, ${narrowRatioDivisor})
        result
        `;
        }

//         const pythonCode = `
// from PIL import Image, ImageFilter, ImageDraw
// from io import BytesIO
// import base64

// def raster_blur_effect(data, file_type, slice_count, blur_radius, highlight_intensity, shadow_intensity):
//     input_image = Image.open(BytesIO(base64.b64decode(data)))
//     width, height = input_image.size

//     num_slices = (slice_count + 1) // 2
//     original_slice_width = int(width / num_slices)
//     step_width = int(width / (num_slices / 0.5))
//     compressed_slice_width = int(width / slice_count)

//     slices = []
//     for i in range(slice_count):
//         start_x = int(i * step_width)
//         end_x = min(start_x + original_slice_width, width)
//         slice_strip = input_image.crop((start_x, 0, end_x, height))

//         if blur_radius > 0:
//             slice_strip = slice_strip.filter(ImageFilter.GaussianBlur(blur_radius))

//         if highlight_intensity > 0:
//             highlight_width = int(original_slice_width * 0.2)
//             gradient = Image.new('RGBA', (highlight_width, height), (255, 255, 255, 0))
//             gradient_draw = ImageDraw.Draw(gradient)
//             for x in range(highlight_width):
//                 alpha = int(255 * highlight_intensity * (1 - x / highlight_width))
//                 gradient_draw.line([(x, 0), (x, height)], fill=(255, 255, 255, alpha))
//             slice_strip.paste(gradient, (0, 0), gradient)

//         if shadow_intensity > 0:
//             shadow_width = int(original_slice_width * 0.1)
//             gradient = Image.new('RGBA', (shadow_width, height), (0, 0, 0, 0))
//             gradient_draw = ImageDraw.Draw(gradient)
//             for x in range(shadow_width):
//                 alpha = int(255 * shadow_intensity * (x / shadow_width))
//                 gradient_draw.line([(x, 0), (x, height)], fill=(0, 0, 0, alpha))
//             slice_strip.paste(gradient, (original_slice_width - shadow_width, 0), gradient)

//         slices.append(slice_strip.resize((compressed_slice_width, height), Image.Resampling.LANCZOS))

//     final_image_width = compressed_slice_width * slice_count
//     final_image = Image.new('RGB', (final_image_width, height))
//     x_offset = 0
//     for strip in slices:
//         final_image.paste(strip, (x_offset, 0))
//         x_offset += compressed_slice_width

//     final_image = final_image.resize((width, height), Image.Resampling.LANCZOS)
//     output_buffer = BytesIO()
//     final_image.save(output_buffer, format="PNG")
//     return base64.b64encode(output_buffer.getvalue()).decode("utf-8")

// result = raster_blur_effect("${uploadedImage.split(",")[1]}", "PNG", ${sliceCount}, ${blurRadius}, ${highlightIntensity}, ${shadowIntensity})
// result
// `;

        try {
            const result = await pyodide.runPythonAsync(pythonCode);
            processedImage = `data:image/png;base64,${result}`;
            isDownloadable = true;
        } catch (error) {
            console.error("Python execution error:", error);
            alert("An error occurred during processing. Check the console for details.");
        } finally {
            isLoading = false; // 加载完成，隐藏动画
        }
    };

    const handleDownload = () => {
        if (!processedImage) {
            alert("No processed image available for download.");
            return;
        }

        // 确定下载的图片格式
        const isPng = processedImage.startsWith('data:image/png');
        const extension = isPng ? 'png' : 'jpg';

        const baseName = originalFileName ? originalFileName.split('.')[0] : 'image';
        const downloadName = `${baseName}_rasterization.${extension}`;

        // 创建一个链接用于下载
        const link = document.createElement('a');
        link.href = processedImage;
        link.download = downloadName;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    };

    // 备份onmount函数
    // onMount(async () => {
    //     updateDimensions();
    //     window.addEventListener('resize', updateDimensions);

    //     try {
    //         await loadScript('https://cdn.jsdelivr.net/pyodide/v0.27.5/full/pyodide.js'); // 0.23.4
    //         await loadPyodideAndPackages();
    //     } catch (error) {
    //         console.error("Failed to load Pyodide:", error);
    //     }

    //     return () => {
    //         window.removeEventListener('resize', updateDimensions);
    //     };
    // });

    onMount(async () => {
        updateDimensions();
        window.addEventListener('resize', updateDimensions);

        try {
            await loadScript('pyodide.js');
            await loadPyodideAndPackages();
        } catch (error) {
            console.error("Failed to load Pyodide:", error);
        }

        return () => {
            window.removeEventListener('resize', updateDimensions);
        };
    });
</script>


<main>
    <div class="mode-selector">
        <label>
            {isEnglish ? "Effect Mode" : "效果模式"}:
            <select bind:value={mode} on:change={handleModeChange}>
                <option value="raster">{isEnglish ? "Raster Glass Effect" : "光栅玻璃效果"}</option>
                <option value="rainbow">{isEnglish ? "Rainbow Glass Effect" : "虹玻璃效果"}</option>
            </select>
        </label>
    </div>
    
    <!-- Left Rectangle -->
    <div class="rectangle left">
        <div class="label">{isEnglish ? "Selected Image" : "已选图片"}</div>
        <img class="uploaded-image" src={uploadedImage} alt="Uploaded" />
    </div>
    <button class="upload-button" on:click={() => document.getElementById('image-upload').click()}>{isEnglish ? "Choose Image" : "选择图片"}</button>
    <input type="file" accept="image/*" id="image-upload" style="display: none" on:change={handleImageUpload} />

    <!-- Right Rectangle -->
    <div class="rectangle right">
        <div class="label">{isEnglish ? "Rasterized Image" : "光栅化图片"}</div>
        <img class="processed-image" src={processedImage} alt="Processed" />
    </div>

    <!-- Center Rectangle -->
    <div class="rectangle center">
        <div class="label">{isEnglish ? "Parameter Adjustment" : "参数调节"}</div>
        <div class="controls">
            <label>{isEnglish ? "Slice Count" : "切片数量"}: {sliceCount}
                <input type="range" min="5" max="100" bind:value={sliceCount} />
            </label>
            <label>{isEnglish ? "Blur Radius" : "模糊半径"}: {blurRadius}
                <input type="range" min="0" max="100" bind:value={blurRadius} />
            </label>
            <label>{isEnglish ? "Highlight" : "光照强度"}: {highlightIntensity.toFixed(2)}
                <input type="range" min="0" max="1.0" step="0.01" bind:value={highlightIntensity} />
            </label>
            <label>{isEnglish ? "Shadow" : "阴影强度"}: {shadowIntensity.toFixed(2)}
                <input type="range" min="0" max="1.0" step="0.01" bind:value={shadowIntensity} />
            </label>
            {#if mode === "rainbow"}
                <label>{isEnglish ? "Compression Ratio" : "聚光压缩比"}: {narrowRatioDivisor}
                    <input type="range" min="2" max="20" bind:value={narrowRatioDivisor} />
                </label>
            {/if}
        </div>
        <button class="convert-button" on:click={handleConversion}>{isEnglish ? "Convert" : "转换"}</button>
    </div>

    <div class="loading" class:hidden={!isLoading}>
        <img src="icons8-loading4.gif" alt={isEnglish ? "Loading" : "加载中..."} />
    </div>

    <button class="download-button" on:click={handleDownload} disabled={!isDownloadable}>{isEnglish ? "Download" : "下载"}</button>

    <!-- Language Toggle -->
    <div class="language-toggle">
        <span>中文</span>
        <label class="switch">
            <input type="checkbox" bind:checked={isEnglish} />
            <span class="slider"></span>
        </label>
        <span>English</span>
    </div>
</main>

<style>
    :global(body) {
        margin: 0;
        overflow: hidden;
        background-image: url('/bg7.png'); 
        background-size: cover;
        background-position: center;
        background-repeat: no-repeat;
    }

    main {
        position: relative;
        width: 100vw;
        height: 100vh;
        display: flex;
        justify-content: center;
        align-items: center;
        overflow: hidden;
    }

    .rectangle {
        position: absolute;
        background-color: rgba(248, 248, 248, 0.9);
        border-radius: 10px;
        box-shadow: 0 4px 9px rgba(0, 0, 0, 0.13);
        display: flex;
        justify-content: center;
        align-items: center;
        flex-direction: column;
    }

    .label {
        position: absolute;
        top: -10%;
        font-size: 1.45vw;
        color: #333;
        text-align: center;
    }

    .left {
        width: 35%;
        height: 70%;
        left: 3%;
        top: 50%;
        transform: translateY(-50%);
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    .right {
        width: 35%;
        height: 70%;
        right: 3%;
        top: 50%;
        transform: translateY(-50%);
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    .center {
        width: 18%;
        height: 50%;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        display: flex;
        flex-direction: column;
        justify-content: space-around;
    }

    .uploaded-image {
        max-width: 95%;
        max-height: 90%;
        margin-bottom: 10px;
        border: 1px solid #ccc;
        border-radius: 5px;
    }

    .processed-image {
        max-width: 95%;
        max-height: 90%;
        margin-top: 10px;
        border: 1px solid #ccc;
        border-radius: 5px;
    }

    .upload-button {
        position: absolute;
        left: 15%;
        top: calc(50% + 40%);
        transform: translateY(-50%);
        background-color: #007bff;
        color: white;
        border: none;
        border-radius: 2vw;
        padding: 0.5% 1%;
        font-size: 1vw;
        cursor: pointer;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
        transition: background-color 0.3s;
        min-width: 10%;
        min-height: 5%;
    }

    .upload-button:hover {
        background-color: #0056b3;
    }

    .convert-button {
        background-color: #28a745;
        color: white;
        border: none;
        border-radius: 2vw;
        padding: 3% 8%;
        font-size: 1vw;
        cursor: pointer;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
        transition: background-color 0.3s;
        min-width: 40%;
        min-height: 10%;
    }

    .convert-button:hover {
        background-color: #218838;
    }

    .controls {
        display: flex;
        flex-direction: column; /* 垂直布局 */
        justify-content: space-between;
        align-items: center; /* 保持子元素居中 */
        gap: 16px; /* 控件之间的间距 */
    }

    .controls label {
        display: flex;
        flex-direction: column; /* 垂直排列文字和拖动条 */
        align-items: center; /* 居中对齐 */
        font-size: 0.95rem;
        gap: 12px; /* 文字与拖动条之间的间距 */
        width: 300px; /* 固定宽度 */
    }

    .controls input[type="range"] {
        width: 50%; /* 宽度占满父容器 */
        max-width: none; /* 移除最大宽度限制 */
        margin: 0 auto;
        appearance: none; /* 移除浏览器默认样式 */
        height: 6px; /* 自定义高度 */
        background: #ddd; /* 背景颜色 */
        border-radius: 5px; /* 圆角 */
        outline: none; /* 去除聚焦时的轮廓 */
        transition: background-color 0.3s;
    }

    .controls input[type="range"]::-webkit-slider-thumb {
        appearance: none; /* 移除默认样式 */
        width: 16px; /* 滑块宽度 */
        height: 16px; /* 滑块高度 */
        background: #007bff; /* 滑块颜色 */
        border-radius: 50%; /* 圆形滑块 */
        cursor: pointer;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2); /* 阴影 */
        transition: background-color 0.3s;
    }

    .controls input[type="range"]::-moz-range-thumb,
    .controls input[type="range"]::-ms-thumb {
        width: 16px;
        height: 16px;
        background: #007bff;
        border-radius: 50%;
        cursor: pointer;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
    }

    .loading {
        position: absolute;
        top: 80%; /* 在参数调节框下方 */
        left: 50%; /* 水平居中 */
        transform: translateX(-50%);
        width: 50px; /* 设置动画宽度 */
        height: 50px; /* 设置动画高度 */
        display: flex;
        justify-content: center;
        align-items: center;
    }

    .hidden {
        display: none;
    }

    .download-button {
        position: absolute;
        right: 15%;
        top: calc(50% + 40%);
        transform: translateY(-50%);
        background-color: #007bff;
        color: white;
        border: none;
        border-radius: 2vw;
        padding: 0.5% 1%;
        font-size: 1vw;
        cursor: pointer;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
        transition: background-color 0.3s;
        min-width: 10%;
        min-height: 5%;
    }

    .download-button:disabled {
        background-color: #ccc; /* 灰色背景 */
        cursor: not-allowed; /* 禁用状态的鼠标样式 */
    }

    .download-button:hover:enabled {
        background-color: #0056b3; /* 仅在启用时生效 */
    }

    .language-toggle {
        position: absolute;
        top: 2%;
        right: 1%;
        display: flex;
        align-items: center;
        gap: 10px;
        font-size: 1rem;
    }

    .switch {
        position: relative;
        display: inline-block;
        width: 40px;
        height: 20px;
    }

    .switch input {
        opacity: 0;
        width: 0;
        height: 0;
    }

    .slider {
        position: absolute;
        cursor: pointer;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: #e0aaaa;
        transition: 0.4s;
        border-radius: 20px;
    }

    .slider:before {
        position: absolute;
        content: "";
        height: 16px;
        width: 16px;
        left: 2px;
        bottom: 2px;
        background-color: white;
        transition: 0.4s;
        border-radius: 50%;
    }

    input:checked + .slider {
        background-color: #626ea5;
    }

    input:checked + .slider:before {
        transform: translateX(20px);
    }

    .mode-selector {
        position: absolute;
        top: 2%;
        left: 50%;
        transform: translateX(-50%);
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 1.05rem;
        z-index: 10;
        background-color: rgba(255, 255, 255, 0.85);
        padding: 6px 16px;
        border-radius: 8px;
        box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    }

    .mode-selector select {
        margin-left: 10px;
        font-size: 0.95rem;
        padding: 4px 8px;
        border-radius: 6px;
        border: 1px solid #ccc;
        background-color: white;
    }

</style>