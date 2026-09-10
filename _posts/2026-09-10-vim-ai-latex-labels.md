---
layout: post
title: Let Vim Ask an AI to Name Your LaTeX Labels
date: 2026-09-10 08:00:00 +0800
description: A Vim9 function that sends the current line and its surrounding context to an AI model and inserts a sensible \label{...} for you — one keystroke, no more naming by hand.
tags: tools
categories: blog
---

When writing LaTeX, I constantly need `\label{...}` for equations, theorems, lemmas, and sections — and I constantly waste time inventing names like `eq:adjunction-unit` by hand. So I wired Vim to an AI model: put the cursor on the line, press `<leader>lj`, and a context-aware label appears automatically.

## What `LatexInsertLabel` does

The function lives in my `~/.vimrc` (it uses Vim9 script syntax, so it needs **Vim 9**). In one call it:

1. **Builds a prompt.** It reads a small instruction file (`~/.vim/skills/latex-label.md`) that tells the model the naming rules, then appends the current line as the `TARGET TO LABEL` plus 12 lines of surrounding context above and below.
2. **Calls the AI.** It first tries a Python helper (`~/.vim/scripts/kimi-label.py`) which streams the reply from the `kimi` CLI and returns as soon as the first assistant message arrives (with a 12-second safety timeout). If the helper is missing or returns nothing, it falls back to calling `kimi --output-format stream-json` directly and parses the JSON lines itself, with a couple of extra regex-based fallbacks.
3. **Sanitizes the answer.** Whatever the model says, it extracts just the label body (the part inside `\label{...}`, or the first label-shaped token).
4. **Inserts it intelligently.** It splits the current line into code and trailing comment (respecting escaped `\%`), then:
   - if the line already has a `\label{...}`, it **replaces** the name in place;
   - if the line contains `\end{...}`, it inserts the label **on the line before**;
   - if the line is blank, it writes the label on that line;
   - otherwise it appends the label to the end of the code part, before the comment.

So pressing `<leader>lj` again on the same line re-names the label instead of duplicating it — handy when the first suggestion is not good enough.

## The function

Here is the complete code, exactly as it sits in my `~/.vimrc`:

```vim
# Insert \label{eq:...} on the current line using Kimi (no need to type \label{})
# Cursor on the equation / theorem line, then <leader>lj
def g:LatexInsertLabel()
  var skillfile = expand('~/.vim/skills/latex-label.md')
  if !filereadable(skillfile)
    echoerr 'Missing ' .. skillfile
    return
  endif
  var skill = readfile(skillfile)->join("\n")
  var cur_line = getline('.')
  var start_line = max([1, line('.') - 12])
  var end_line = min([line('$'), line('.') + 12])
  var ctx = getline(start_line, end_line)->join("\n")
  var prompt = skill .. "\n\nTARGET TO LABEL:\n" .. cur_line .. "\n\nSURROUNDING CONTEXT:\n" .. ctx

  echo 'Naming label...'
  redraw
  var helper = expand('~/.vim/scripts/kimi-label.py')
  var name = ''
  if filereadable(helper)
    name = system(helper, prompt)->trim()
  endif

  if name == ''
    var cmd = 'kimi --output-format stream-json -m kimi-code/kimi-for-coding-highspeed -p ' .. shellescape(prompt)
    var raw = system(cmd)
    if v:shell_error != 0
      echoerr 'kimi failed: ' .. raw
      return
    endif

    for item in split(raw, "[\r\n]\\+")
      if item =~ '^{"role":"assistant"'
        try
          var data = json_decode(item)
          if type(data) == v:t_dict && has_key(data, 'content')
            name = data.content
            break
          endif
        catch
        endtry
      endif
    endfor

    if name == ''
      name = matchstr(raw, '"role":\s*"assistant",\s*"content":\s*"\zs[^"]*\ze"')
    endif

    if name == ''
      for item in reverse(split(raw, "[\r\n]\\+"))
        if item !~? '^\s*\(kimi\|to resume\|error\|{\)' && item =~ '[A-Za-z]'
          name = item
          break
        endif
      endfor
    endif
  endif

  var label_body = matchstr(name, '\\label{\zs[A-Za-z0-9_:.+-]\+\ze}')
  if label_body == ''
    label_body = matchstr(name, '[A-Za-z][A-Za-z0-9_:.+-]*')
  endif
  name = label_body

  if name == ''
    echoerr 'model returned no label'
    return
  endif

  var l = getline('.')
  var comment_idx = match(l, '\(^\|[^\\]\)\(\\\\\)*\zs%')
  var code = ''
  var comment = ''
  if comment_idx >= 0
    code = strpart(l, 0, comment_idx)
    comment = strpart(l, comment_idx)
  else
    code = l
    comment = ''
  endif

  comment = substitute(comment, '\s*\\label{[^}]*}', '', 'g')

  if code =~ '\\label{'
    code = substitute(code, '\\label{\zs[^}]*', name, '')
    setline('.', code .. comment)
    echo 'updated \label{' .. name .. '}'
    return
  endif

  if code =~ '\\end{'
    append(line('.') - 1, matchstr(l, '^\s*') .. '\label{' .. name .. '}')
    echo 'inserted \label{' .. name .. '}'
    return
  endif

  if code =~ '^\s*$'
    if comment != ''
      setline('.', matchstr(code, '^\s*') .. '\label{' .. name .. '} ' .. comment)
    else
      setline('.', matchstr(code, '^\s*') .. '\label{' .. name .. '}')
    endif
    echo 'inserted \label{' .. name .. '}'
    return
  endif

  code = substitute(code, '\s*$', '', '')
  var sep = (code =~ '[A-Za-z0-9]$') ? ' ' : ''
  code = code .. sep .. '\label{' .. name .. '}'

  setline('.', code .. comment)
  echo 'inserted \label{' .. name .. '}'
enddef
augroup latex_insert_label
  autocmd!
  autocmd FileType tex nnoremap <buffer> <leader>lj <ScriptCmd>g:LatexInsertLabel()<CR>
augroup END
```

## Where to put it

The function itself goes into your `~/.vimrc`, anywhere at the top level (the `def g:` form and `<ScriptCmd>` mapping require Vim 9). The two helper files it calls live under `~/.vim/`:

- `~/.vim/skills/latex-label.md` — the prompt that defines the naming rules;
- `~/.vim/scripts/kimi-label.py` — the streaming helper script (make it executable: `chmod +x`).

You also need the [`kimi` CLI](https://www.kimi.com/code) installed and logged in, since that is what actually talks to the model.

## The prompt file

`~/.vim/skills/latex-label.md` is deliberately terse — short prompts answer faster and more predictably:

```markdown
You invent a LaTeX \label name from the nearby source.

Rules:
- Reply with ONLY the label body, e.g. eq:adjunction
- No backslash, no \label, no braces, no quotes, no extra words
- Do not use tools or edit files
- Prefixes: eq: thm: lem: prop: cor: def: sec: fig:
- Short, stable, ASCII: English or standard math tokens
- Do not copy an existing label from the excerpt
- Reply immediately without thinking or explanation
```

Because the context window around the cursor is included, the model tends to pick up the actual theorem or equation content — e.g. `\label{thm:serre-vanishing}` next to a statement about Serre vanishing — instead of a generic name.

## The helper script

`~/.vim/scripts/kimi-label.py` reads the prompt from its argument or stdin, launches the CLI, and returns the label as soon as the first assistant message streams in (killing the process afterwards so Vim does not wait for the full session to end):

```python
#!/usr/bin/env python3
import sys, subprocess, json, re, time

def main():
    if len(sys.argv) > 1:
        prompt = sys.argv[1]
    else:
        prompt = sys.stdin.read()
    if not prompt or not prompt.strip():
        return

    cmd = [
        "kimi", "--output-format", "stream-json",
        "-m", "kimi-code/kimi-for-coding-highspeed",
        "-p", prompt.strip()
    ]

    try:
        p = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.DEVNULL, text=True)
    except Exception as e:
        sys.stderr.write(f"Error starting kimi: {e}\n")
        return

    start = time.time()
    try:
        for line in p.stdout:
            # safety timeout: 12 seconds max
            if time.time() - start > 12:
                break
            if '"role":"assistant"' in line:
                try:
                    data = json.loads(line)
                    content = data.get("content", "").strip()
                    m = re.search(r'\\label\{([^}]+)\}', content)
                    if m:
                        print(m.group(1))
                    else:
                        m_word = re.search(r'[A-Za-z][A-Za-z0-9_:.+-]*', content)
                        if m_word:
                            print(m_word.group(0))
                        else:
                            print(content)
                except Exception:
                    print(line.strip())
                try:
                    p.kill()
                except Exception:
                    pass
                return
    except Exception as e:
        sys.stderr.write(str(e) + "\n")
    finally:
        try:
            p.kill()
        except Exception:
            pass

if __name__ == "__main__":
    main()
```

## Notes

- The whole round trip takes about a second with the high-speed model, which is much faster than me deliberating over `prop:finite-type-base-change` vs `prop:bc-finite-type`.
- The fallback path inside the Vim function means the mapping still works even if the Python helper is missing — it is just slower because it waits for the CLI to exit.
- The same pattern generalizes: any "look at the cursor's neighborhood and do something small" task (generating BibTeX keys, caption text, proof remarks) can reuse this skeleton with a different skill file.
