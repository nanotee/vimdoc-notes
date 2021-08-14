# Vimdoc notes

## Introduction

The recent stream celebrating the release of Neovim 0.5 briefly touched on Vim help files and echoed a lot of my sentiments regarding the format. I figured I might as well publish this since I'm not the only one who's interested in exploring possibilities for evolving/replacing the format.

The way I see it, the Vimdoc format is limited in the following ways:
- There's no spec or authoritative document describing the format. `:help help-writing` is a start, but many of the syntax groups are undocumented
- There's not a lot of tooling around it. This makes it a pain to generate HTML from Vimdoc and to automate documentation generation (I wrote a bunch of brittle scripts for nvim-lua-guide, and the experience was not pleasant)
- Since there's no spec, some syntax rules are only enforced through convention. It's not that uncommon for Vim help documents to contain tabular data, but since there are no syntax rules for tables, writers are free to chose any format. This makes it harder to build tools for Vimdocs.
- It lacks features that I would consider "nice to have", like syntax highlighting in code blocks

The one feature I like is that you can generate tags for arbitrary locations in a document, which makes documentation very discoverable. If another format is chosen to replace it, you would have to be able to elegantly add metadata to generate tags

## Potential candidates for replacing Vimdoc

I basically only know markdown

### Markdown

Pros:
- Comprehensive spec (CommonMark)
- Basically supported universally
- You get a ton of tools for free, it's really easy to generate HTML
- LSP and `:checkhealth` already use Markdown, making Markdown "first-class" in Neovim is definitely an idea I'm interested in

Cons:
- Apparently it's hard to make a tree-sitter grammar for it
- How do you handle tags? Syntax extension via [markdown-it](https://github.com/markdown-it/markdown-it)?
- How do you handle links to tags? Maybe a custom URL scheme like `tag://user-manual`? Syntax extension?
- It's not uncommon for help documents to contain arbitrary indentation to make things look nicer. This is not possible in markdown because it gets interpreted as a code block. Maybe that's a job for anticonceal instead of hard-coding it?

### AsciiDoc

TODO

### reStructuredText

TODO

### org-mode (hah!)

TODO

## Specification

It's incomplete and I have no idea how to write a specification document. You've been warned.

### "Official" groups

#### Sections

Section delimiter: at least 6 `=` or `-` signs (usually 78) taking up an entire line  
The section delimiter accepts any character as long as at least 3 `=` or `-` are on either side of the text

Followed by the headline on the next line with a help tag aligned to the right of the headline

```help
==============================================================================
Headline                                                             *headline*

==================================headline====================================
Headline                                                             *headline*

------------------------------------------------------------------------------
Headline                                                             *headline*

----------------------------------headline------------------------------------
Headline                                                             *headline*
```

Syntax definition:

```vim
syn match helpHeadline		"^[-A-Z .][-A-Z0-9 .()_]*[ \t]\+\*"me=e-1
syn match helpSectionDelim	"^===.*===$"
syn match helpSectionDelim	"^---.*--$"
```

#### Column heading

Any sequence of characters followed by a tilde at the end of the line

OR

Sequence of uppercase characters followed by a right-aligned help tag  
(Note: the highlight group is `helpHeadline` but `gO` indents it, suggesting its role is more akin to that of a column heading)

OR

Sequence of uppercase characters on a single line  
(Note: no highlight group but `gO` picks it up)

```help
Column heading~

COLUMN HEADING						*column-heading*

COLUMN HEADING
```

Syntax definition:

```vim
syn match helpHeader		"\s*\zs.\{-}\ze\s\=\~$" nextgroup=helpIgnore
```

#### Code block

Start the code block with a `>` character at the end of the line before the block  
End the code block with a `<` character at the start of the line after the block

The code block has to be indented by at least one space or tab

Any line starting at column 1 implicitly ends the code block without needing a `<` character

```help
Eg: >
    function! s:example_codeblock() abort
        echo 'blah'
    endfunction
<

>
    function! s:example_codeblock() abort
        echo 'blah'
    endfunction
blah
```

Syntax definition:

```vim
syn region helpExample	matchgroup=helpIgnore start=" >$" start="^>$" end="^[^ \t]"me=e-1 end="^<" concealends
```

#### Help tags

Any sequence of characters except whitespace enclosed in `*` characters. The closing `*` should be followed by whitespace or a newline  
Often found in front of section headlines and column headings

Used to generate help tags with the `:helptags` command

```help
*tag-name*
```

Syntax definition:

```vim
syn match helpStar		contained "\*" conceal
syn match helpHyperTextEntry	"\*[#-)!+-~]\+\*\s"he=e-1 contains=helpStar
syn match helpHyperTextEntry	"\*[#-)!+-~]\+\*$" contains=helpStar
```

#### Hot-link

##### Normal

```help
|tag-name|
```

Syntax definition:

```vim
syn match helpBar		contained "|" conceal
syn match helpHyperTextJump	"\\\@<!|[#-)!+-~]\+|" contains=helpBar
```

##### Options

The only characters allowed are lowercase alphabetic (ASCII-only) characters OR `t_` followed by exactly two characters (any)

```help
'tagname'
't_$*'
```

Syntax definition:

```vim
syn match helpOption		"'[a-z]\{2,\}'"
syn match helpOption		"'t_..'"
```

##### Commands

```help
`:tagname`
```

Syntax definition:

```vim
syn match helpBacktick	contained "`" conceal
syn match helpCommand		"`[^` \t]\+`"hs=s+1,he=e-1 contains=helpBacktick
syn match helpCommand		"\(^\|[^a-z"[]\)\zs`[^`]\+`\ze\([^a-z\t."']\|$\)"hs=s+1,he=e-1 contains=helpBacktick
```

#### Special

##### Keycodes

```help
<Esc>
<Enter>
<S-Right>
```

Syntax definition:

```vim
syn match helpSpecial		"<[-a-zA-Z0-9_]\+>"
syn match helpSpecial		"<[SCM]-.>"
```

##### Chords

```help
CTRL-O
CTRL-SHIFT-A
META-U
ALT-J
```

Syntax definition:

```vim
syn match helpSpecial		"CTRL-."
syn match helpSpecial		"CTRL-SHIFT-."
syn match helpSpecial		"CTRL-Break"
syn match helpSpecial		"CTRL-PageUp"
syn match helpSpecial		"CTRL-PageDown"
syn match helpSpecial		"CTRL-Insert"
syn match helpSpecial		"CTRL-Del"
syn match helpSpecial		"CTRL-{char}"
syn match helpSpecial		"META-."
syn match helpSpecial		"ALT-."
```

#### Eval

##### Command modifiers

```help
:[count]command
:command [++opt]
```

Syntax definition:

```vim
syn match helpSpecial		"\[N]"
syn match helpSpecial		"\[range]"
syn match helpSpecial		"\[line]"
syn match helpSpecial		"\[count]"
syn match helpSpecial		"\[offset]"
syn match helpSpecial		"\[cmd]"
syn match helpSpecial		"\[num]"
syn match helpSpecial		"\[+num]"
syn match helpSpecial		"\[-num]"
syn match helpSpecial		"\[+cmd]"
syn match helpSpecial		"\[++opt]"
syn match helpSpecial		"\[arg]"
syn match helpSpecial		"\[arguments]"
syn match helpSpecial		"\[ident]"
syn match helpSpecial		"\[addr]"
syn match helpSpecial		"\[group]"
```

##### Parameters

```help
:command {arg1} {arg2}
func({arg1}, {arg2})
```

Syntax definition:

```vim
syn match helpSpecial		"{[-_a-zA-Z0-9'"*+/:%#=[\]<>.,]\+}"
```

##### Optional

Function arguments or command tails surrounded with `[]` brackets. No highlight group but it's used consistently in `:help`

```help
func({arg1}, {arg2} [, {optionalarg}])
:longcommandwithop[tionaltail][!]
```

#### URL

```help
https://www.vim.org/
https://neovim.io/
mailto:user@host.com
file:/etc/os-release
```

Syntax definition:

```vim
syn match helpURL `\v<(((https?|ftp|gopher)://|(mailto|file|news):)[^' 	<>"]+|(www|web|w3)[a-z0-9_-]*\.[a-z0-9._-]+\.[^' 	<>"]+)[a-zA-Z0-9/]`
```

#### Notices

##### Note

```help
NOTE: info
Note something
```

Syntax definition:

```vim
syn keyword helpNote		note Note NOTE note: Note: NOTE: Notes Notes:
```

##### Warning

```help
WARNING: info
```

Syntax definition:

```vim
syn keyword helpWarning		WARNING WARNING: Warning:
```

##### Deprecated

```help
DEPRECATED: use X instead
```

Syntax definition:

```vim
syn keyword helpDeprecated	DEPRECATED DEPRECATED: Deprecated:
```

#### Highlight groups

```help
	 Todo	add more stuff
	*Error	scary error
```

Syntax definition:

```vim
syn match helpComment		"\t[* ]Comment\t\+[a-z].*"
syn match helpConstant		"\t[* ]Constant\t\+[a-z].*"
syn match helpString		"\t[* ]String\t\+[a-z].*"
syn match helpCharacter		"\t[* ]Character\t\+[a-z].*"
syn match helpNumber		"\t[* ]Number\t\+[a-z].*"
syn match helpBoolean		"\t[* ]Boolean\t\+[a-z].*"
syn match helpFloat		"\t[* ]Float\t\+[a-z].*"
syn match helpIdentifier	"\t[* ]Identifier\t\+[a-z].*"
syn match helpFunction		"\t[* ]Function\t\+[a-z].*"
syn match helpStatement		"\t[* ]Statement\t\+[a-z].*"
syn match helpConditional	"\t[* ]Conditional\t\+[a-z].*"
syn match helpRepeat		"\t[* ]Repeat\t\+[a-z].*"
syn match helpLabel		"\t[* ]Label\t\+[a-z].*"
syn match helpOperator		"\t[* ]Operator\t\+["a-z].*"
syn match helpKeyword		"\t[* ]Keyword\t\+[a-z].*"
syn match helpException		"\t[* ]Exception\t\+[a-z].*"
syn match helpPreProc		"\t[* ]PreProc\t\+[a-z].*"
syn match helpInclude		"\t[* ]Include\t\+[a-z].*"
syn match helpDefine		"\t[* ]Define\t\+[a-z].*"
syn match helpMacro		"\t[* ]Macro\t\+[a-z].*"
syn match helpPreCondit		"\t[* ]PreCondit\t\+[a-z].*"
syn match helpType		"\t[* ]Type\t\+[a-z].*"
syn match helpStorageClass	"\t[* ]StorageClass\t\+[a-z].*"
syn match helpStructure		"\t[* ]Structure\t\+[a-z].*"
syn match helpTypedef		"\t[* ]Typedef\t\+[Aa-z].*"
syn match helpSpecial		"\t[* ]Special\t\+[a-z].*"
syn match helpSpecialChar	"\t[* ]SpecialChar\t\+[a-z].*"
syn match helpTag		"\t[* ]Tag\t\+[a-z].*"
syn match helpDelimiter		"\t[* ]Delimiter\t\+[a-z].*"
syn match helpSpecialComment	"\t[* ]SpecialComment\t\+[a-z].*"
syn match helpDebug		"\t[* ]Debug\t\+[a-z].*"
syn match helpUnderlined	"\t[* ]Underlined\t\+[a-z].*"
syn match helpError		"\t[* ]Error\t\+[a-z].*"
syn match helpTodo		"\t[* ]Todo\t\+[a-z].*"
```

### Other stuff found in the wild

#### Shell commands

```help
% mkdir -p test
% cd test
```

[Reference](https://github.com/neovim/neovim/blob/6f0d4ccc43d7c8f5dd72d0d83e54e947d05117fa/runtime/doc/repeat.txt#L502-L504)

#### Lists

##### Unordered

```help
- Item 1
- Item 2
- Item 3
```

[Reference](https://github.com/neovim/neovim/blob/6f0d4ccc43d7c8f5dd72d0d83e54e947d05117fa/runtime/doc/options.txt#L830-L859)

##### Ordered

```help
1) Item 1
2) Item 2
3) Item 3
```

[Reference](https://github.com/neovim/neovim/blob/6f0d4ccc43d7c8f5dd72d0d83e54e947d05117fa/runtime/doc/editing.txt#L1483-L1567)


#### Tables

Style 1:

```help
header1          header2 ~
foo              barbazquux
verylongword     test
```

[Reference 1](https://github.com/neovim/neovim/blob/6f0d4ccc43d7c8f5dd72d0d83e54e947d05117fa/runtime/doc/options.txt#L2582-L2595)
[Reference 2](https://github.com/tpope/vim-fugitive/blob/6c53da0783a15d2dcde504ae299468ac69078ebe/doc/fugitive.txt#L571-L593)

Style 2:

These are similar to markdown/orgmode tables.
Has to start with whitespace, otherwise row delimiters are highlighted as headers

```help
 -------------+-----------
 header1      | header2  ~
 -------------+-----------
 foo          | barbazquux
 verylongword | test
 -------------+-----------
```

[Reference](https://github.com/junegunn/fzf.vim/blob/e34f6c129d39b90db44df1107c8b7dfacfd18946/doc/fzf-vim.txt#L115-L142)
