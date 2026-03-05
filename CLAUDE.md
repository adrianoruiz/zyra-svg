# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Image-to-SVG converter built with Vue 3 and Vite. Users upload raster images (JPG/PNG/WebP) which are converted to SVG vectors using `imagetracerjs`. The UI is in Portuguese (pt-BR).

## Commands

- `npm run dev` — Start dev server with HMR
- `npm run build` — Production build to `dist/`
- `npm run preview` — Preview production build locally

## Architecture

Single-page Vue 3 app with minimal structure:

- `src/App.vue` — Main component containing all app logic: file upload, image tracing via `imagetracerjs`, SVG preview, and download
- `src/main.js` — App entry point
- `src/components/HelloWorld.vue` — Default template component (unused by App.vue)
- `vite.config.js` — Vite config with Vue plugin

## Key Dependencies

- **Vue 3** (`<script setup>` SFCs, Composition API with `ref`)
- **imagetracerjs** — Raster-to-SVG conversion library (CommonJS module, requires default export handling in ESM context)
- **Vite 7** with `@vitejs/plugin-vue`

## Notes

- The `imagetracerjs` import uses a CommonJS compatibility pattern: `ImageTracerModule.default || ImageTracerModule`
- SVG output is rendered via `v-html` directive
- Download uses `URL.createObjectURL` with proper cleanup to avoid memory leaks
