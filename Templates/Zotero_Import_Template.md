# [{{title}}]({{desktopURI}})

 **Authors:** {{authors}}
 **Published:** {{date | format("YYYY")}}
 **Journal/Conference:** {{ publicationTitle or proceedingsTitle or bookTitle or university or institution }}
**Citekey:** `{{citekey}}`

---
## Abstract
> [!abstract]
>  ```
>  {{abstractNote}}
>  ```

---

{% persist "notes" %}
{%- if isFirstImport %}
## Summary
<!-- claude-summary-start -->
*Pending*
<!-- claude-summary-end -->
{%- endif %}
{% endpersist %}

## Notes & Highlights

{%- persist "obsidian-freeform-notes" %}
{%- if isFirstImport %}

### Freeform Notes

{%- endif %}
{%- endpersist %}

### From Zotero

{%- for annotation in annotations %}

---
{%- set callout = "quote" -%}
{%- if annotation.colorCategory == "Red" %}{%- set callout = "danger" -%}
{%- elif annotation.colorCategory == "Orange" %}{%- set callout = "warning" -%}
{%- elif annotation.colorCategory == "Yellow" %}{%- set callout = "note" -%}
{%- elif annotation.colorCategory == "Green" %}{%- set callout = "success" -%}
{%- elif annotation.colorCategory == "Blue" %}{%- set callout = "info" -%}
{%- elif annotation.colorCategory == "Purple" %}{%- set callout = "example" -%}
{%- elif annotation.colorCategory == "Magenta" %}{%- set callout = "question" -%}
{%- elif annotation.colorCategory == "Gray" %}{%- set callout = "quote" -%}
{%- endif %}
{%- if annotation.annotatedText %}
> [!{{callout}}] Highlight (Page {{annotation.pageLabel}})
> {{annotation.annotatedText}}
{%- endif %}
{%- if annotation.imageRelativePath %}

![[{{annotation.imageRelativePath}}]]
{%- endif %}
{%- if annotation.comment %}
- **My Note:** {{annotation.comment}}
{%- endif %}
{%- persist annotation.id %}{%- endpersist %}
{%- endfor %}