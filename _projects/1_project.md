---
layout: page
title: Recording homework submissions from cover photos
description: Photograph a pile of student notebooks; OCR on the covers records every student ID and name automatically.
importance: 1
category: tools
---

Students hand in homework written in paper notebooks, with their **name and student ID written on the cover**. Checking who has submitted means flipping through the pile and copying names by hand.

This project turns that into a phone-camera workflow: take a photo of each cover (or a whole row of notebooks at once), and a script extracts every `(student ID, name)` pair and appends it to a roster file.

## How it works

1. **Capture** — photos of notebook covers, one or several per frame, taken on a phone.
2. **Detect** — text detection finds the handwritten ID/name regions on each cover.
3. **Recognize** — OCR reads the Chinese names and digit IDs.
4. **Correct** — a confusion map fixes classic OCR mistakes in IDs (`O→0`, `I→1`, `Z→2`, `S→5`, `B→8`, and their Chinese look-alikes 一二三四五六七八), then the ID is validated against the class roster.
5. **Record** — matched `(学号, 姓名)` pairs are written to `students.csv`; anything uncertain is flagged for manual review instead of being guessed.

## Main tools

- **PaddleOCR (PP-OCRv5 server det + rec models)** — handwriting-tolerant text detection and recognition, run on CPU
- **OpenCV & NumPy** — image loading, preprocessing, and cropping the detected regions before recognition
- **Python (csv, re, pathlib)** — roster matching, ID correction, and CLI glue
