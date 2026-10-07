---
layout: home
---
## Andrew Arace

Software Engineering Director, developer, and designer.

### Contact

| platform     | contact                                                 |
|:-------------|:--------------------------------------------------------|
| linkedin     | [Andrew Arace](https://www.linkedin.com/in/andrewarace/)| 
| email        | [andrew.arace@gmail.com](mailto:andrew.arace@gmail.com) | 

### Pages

- [🗃 Recipe Box](/recipes)
- [🔽 Downloads](/downloads)

### Writing

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%B %-d, %Y" }}{% if post.description %}<br><small>{{ post.description }}</small>{% endif %}
{% endfor %}
