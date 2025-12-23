---
layout: default
title: "Postagens"
---

## Postagens

Textos e anotações aleatórias.

{% assign posts = site.posts %}

<table class="lista-postagens">
    <colgroup>
        <col style="width: 30%;">
        <col style="width: 70%;">
    </colgroup>
    <tbody>
        {% for post in posts %}
        {% assign date = post.date | date: "%d/%m/%Y" %}
        <tr>
            <td>
                <div><a href="{{ site.url }}{{ post.url }}">{{ post.title }}</a></div>
                <sub>{{ date }}</sub>
            </td>
            <td>{{ post.excerpt }}</td>
        </tr>
        {% endfor %}
    </tbody>
</table>