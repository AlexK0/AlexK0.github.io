---
permalink: /ll/
---

<style>
html, body {
    margin: 0;
    padding: 0;
    overflow: hidden;
    width: 100%;
    height: 100%;
}
</style>

<div style="width: 100%; height: 100vh; margin: 0; padding: 0; overflow: hidden;">
    <iframe src="/assets/html/language-learner.html" style="width: 100%; height: 100%; border: none;"></iframe>
</div>

<script>
    // Adjust iframe height to fit the content
    window.addEventListener('load', function() {
        var iframe = document.querySelector('iframe');
        iframe.onload = function() {
            iframe.style.height = iframe.contentWindow.document.body.scrollHeight + 'px';
        };
    });
</script>
