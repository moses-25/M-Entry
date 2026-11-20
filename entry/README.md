npm create vite@latest my-app -- --template react-js
cd my-app
npm install

npm install tailwindcss @tailwindcss/vite

### Open your vite.config.ts (or .js)

import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
plugins: [
tailwindcss(),
react(),
],
});

### src/index.css

@import "tailwindcss";

### test it by adding a Tailwind class to your App.jsx

<h1 className="text-3xl font-bold text-blue-500">
  Hello Tailwind
</h1>
