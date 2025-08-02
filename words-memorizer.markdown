---
permalink: /ll/
---

<style>
html, body {
    margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
}

/* Mobile-friendly iframe container */
.iframe-container {
    width: 100%;
    height: 100vh;
    margin: 0;
    padding: 0;
    position: relative;
    overflow: auto;
}

.iframe-container iframe {
    width: 100%;
    height: 100%;
    border: none;
    display: block;
}

/* Mobile optimizations */
@media (max-width: 768px) {
    .iframe-container {
        height: 100vh;
        overflow: auto;
        -webkit-overflow-scrolling: touch; /* Smooth scrolling on iOS */
    }

    .iframe-container iframe {
        min-height: 100vh;
    }
}

@media (max-width: 480px) {
    .iframe-container {
        height: 100vh;
    }
}
</style>

<div class="iframe-container">
    <iframe src="/assets/html/words-memorizer.html"
            title="Words Memorizer Application"
            allow="clipboard-read; clipboard-write"
            loading="lazy"></iframe>
</div>

<script>
    // Mobile-friendly iframe height adjustment
    window.addEventListener('load', function() {
        var iframe = document.querySelector('iframe');
        var container = document.querySelector('.iframe-container');

        function adjustIframeHeight() {
            try {
                // On mobile, let the iframe scroll naturally
                if (window.innerWidth <= 768) {
                    iframe.style.height = '100vh';
                    return;
                }

                // On desktop, try to adjust to content height
                if (iframe.contentWindow && iframe.contentWindow.document) {
                    var contentHeight = iframe.contentWindow.document.body.scrollHeight;
                    if (contentHeight > 0) {
                        iframe.style.height = Math.max(contentHeight, window.innerHeight) + 'px';
                    }
                }
            } catch (e) {
                // Fallback if cross-origin restrictions apply
                iframe.style.height = '100vh';
            }
        }

        iframe.onload = adjustIframeHeight;
        window.addEventListener('resize', adjustIframeHeight);

        // Initial adjustment
        setTimeout(adjustIframeHeight, 100);
    });
</script>
