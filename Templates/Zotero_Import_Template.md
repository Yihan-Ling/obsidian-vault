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

{% persist "notes" %}{% if isFirstImport %}
## Summary

<!-- claude-summary-start -->
*Pending* 
<!-- claude-summary-end -->

{% endif %}
{% endpersist %}

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