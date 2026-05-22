---
title: "{{title}}"
authors: {{authors}}
year: {{date | format("YYYY")}}
tags: [literature-note, {% for t in tags %}{{t.tag}}{% if not loop.last %}, {% endif %}{% endfor %}]
aliases: ["{{citekey}}"]
---

# {{title}}
[Open in Zotero]({{desktopURI}})

## Metadata
- **Authors:** {{authors}}
- **Published:** {{date | format("YYYY-MM-DD")}}
- **Journal/Conference:** {{publicationTitle}}
- **Citekey:** `{{citekey}}`

## Abstract
> {{abstractNote}}

---

## Notes & Highlights

{% for annotation in annotations %}
{% if annotation.annotatedText %}
> [!quote] Highlight (Page {{annotation.pageLabel}})
> {{annotation.annotatedText}}
{% endif %}

{% if annotation.imageRelativePath %}
![[{{annotation.imageRelativePath}}]]
{% endif %}

{% if annotation.comment %}
- **My Note:** {{annotation.comment}}
{% endif %}

{% endfor %}