# 11 TeXmacs Tips
# Tip 1: 中文标点和拼音的Tab变换
```
a tab → ā
a tab tab → á
a tab tab tab → ǎ
a tab tab tab tab → à

< tab → 《
[ tab → 「
[ tab tab → 『
‘ tab → ’
’ tab → ‘
" tab → “
" tab tab → ” 
```

# Tip 2: Convert to PDF using commandline
```
texmacs --convert xxx.tm yyy.pdf --quit
```

# Tip 3: Customize the side toolbar
``` scheme
(show-side-tools 0 #t)
(tm-widget (texmacs-side-tools)
  (vertical
    (hlist (glue #t #f 15 0) (text "Document tree:") (glue #t #f 15 0))
    ---
    (tree-view noop (buffer-tree) (stree->tree '(unused)))))
```

see Ref 2, currently you have to compile TeXmacs to make it work.

## References
1.  https://github.com/texmacs/texmacs/pull/22
2. https://github.com/texmacs/texmacs/pull/28
