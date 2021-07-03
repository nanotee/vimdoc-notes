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

Only characters allowed are lowercase ASCII characters OR `t_` followed by exactly two characters (any)

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

### Other stuff found in the wild

#### Shell commands

```help
% mkdir -p test
% cd test
```

TODO: add more
