# Blazor WebAssembly(WASM) loading progress and compression
Blazor WebAssembly 目前最大的痛點就是第一次載入時間過長，導致使用者體驗不好。再加上預設的樣板中只顯示一個 "Loading" 字樣來表示網頁正在載入中，無法得知目前載入的進度。為了提升使用者體驗，載入時加入進度條(Progress Bar)來表示載入進度，是目前最佳做法。

## Demo
- [Loading progress Demo](https://blazor-wasm-loading.pages.dev/)

![Demo](https://github.com/KacterLin/Blazor.WASM.Loading/blob/main/content/demo.png?raw=true)

## Getting Started
- 修改 index.html
- 加入 Progress Bar 物件
- 加入 JavaScript
- 將 `autostart="false"` 屬性和值新增至 `script` 腳本
```html
<script src="_framework/blazor.webassembly.js" autostart="false"></script>
```

## JavaScript
- 一般進度顯示方法
``` html
<script type="module">
    var i = 0;
    var allResourcesBeingLoaded = [];
    Blazor.start({
        loadBootResource: function (type, name, defaultUri, integrity) {
            if (type == "dotnetjs")
                return defaultUri;
            var fetchResources = fetch(defaultUri, {
                cache: 'no-cache'
            });
            allResourcesBeingLoaded.push(fetchResources);
            fetchResources.then((r) => {
                i++;
                var total = allResourcesBeingLoaded.length;
                var progressbar = document.getElementById('progressbar');
                var progressvalue = document.getElementById('progressvalue');
                var value = parseInt((i * 100.0) / total);
                var pct = value + '%';
                progressbar.style.width = pct;
                progressvalue.innerHTML = pct;
                console.info(i + '/' + total + ' (' + pct + ') ' + name);
            });
            return fetchResources;
        }
    });
</script>
```
- 進度顯示與壓縮

當網站託管在不支援靜態壓縮的服務商時(Ex: Github Pages、Cloudflare Pages)，這時需要稍微修改程式碼，以符合下載壓縮檔並解壓縮的流程。
``` html
<script src="https://cdn.jsdelivr.net/gh/google/brotli/js/decode.js"></script>
<script type="module">
    var i = 0;
    var allResourcesBeingLoaded = [];
    Blazor.start({
        loadBootResource: function (type, name, defaultUri, integrity) {
            if (type !== 'dotnetjs' && location.hostname !== 'localhost') {
                allResourcesBeingLoaded.push(defaultUri);
                return (async function () {
                    const response = await fetch(defaultUri + '.br', { cache: 'no-cache' });
                    if (!response.ok) {
                        throw new Error(response.statusText);
                    }
                    else {
                        i++;
                        var total = allResourcesBeingLoaded.length;
                        var progressbar = document.getElementById('progressbar');
                        var progressvalue = document.getElementById('progressvalue');
                        var value = parseInt((i * 100.0) / total);
                        var pct = value + '%';
                        progressbar.style.width = pct;
                        progressvalue.innerHTML = pct;
                        console.info(i + '/' + total + ' (' + pct + ') ' + name);
                    }
                    const originalResponseBuffer = await response.arrayBuffer();
                    const originalResponseArray = new Int8Array(originalResponseBuffer);
                    const decompressedResponseArray = BrotliDecode(originalResponseArray);
                    const contentType = type === 
                      'dotnetwasm' ? 'application/wasm' : 'application/octet-stream';
                    return new Response(decompressedResponseArray,
                        { headers: { 'content-type': contentType } });
                })();
            }
        }
    });
</script>
```
## Reference
- https://docs.microsoft.com/aspnet/core/blazor/host-and-deploy/webassembly
- https://github.com/dotnet/aspnetcore/issues/25165?utm_source=pocket_mylist
