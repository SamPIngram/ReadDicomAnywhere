# Read DICOM Anywhere

<img src="images/RDA.png" alt="logo" width="300" height="300">

[Browser based tool](https://SamPIngram.github.io/ReadDicomAnywhere.) viewing/editing DICOM data.

This tool runs in your browser using [Pyodide](https://github.com/iodide-project/pyodide).[Svelte](https://svelte.dev/) is used to implement the user interface, [plotly](https://plotly.com/javascript/) is used for plotting, and [sheetjs](https://github.com/SheetJS/sheetjs) is used for spreadsheet file I/O.

## Build Instructions (for developers)
### [npm](https://www.npmjs.com/)
``` bash
# Install dependencies
npm install

# Server with hot reload at localhost:5000
npm run dev

# Build for production with minification
npm run build

# Deploy public folder to GitHub pages
npm run deploy
```
## License
[MIT](LICENSE)
