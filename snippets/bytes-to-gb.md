Interested in converting bytes to GB, MB, KB? Here's how you can do it in Javascript:

```javascript
function formatBytes (bytes, decimals) {
  if (bytes === 0) return '0 GB'
  if (isNaN(parseInt(bytes))) return bytes
  if (typeof bytes === 'string') bytes = parseInt(bytes)
  if (bytes === 0) return '0';
  const k = 1000;
  const dm = decimals + 1 || 3;
  const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return `${parseFloat((bytes / k ** i).toFixed(dm))} ${sizes[i]}`;
}

// usage
const storage = formatBytes(1000000000) // => '1 GB'
```