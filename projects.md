---
layout: default
title: "Projects"
---

# Projects

<ul class="project-list">
  {% for project in site.data.projects %}
    <li class="project-card">
      {% if project.image and project.image != "" %}
        <a class="project-thumb" href="{{ project.link }}">
          <img src="{{ project.image }}" alt="{{ project.title }}">
        </a>
      {% endif %}
      <div class="project-meta">
        <h3>
          {% if project.link and project.link != "" %}
            <a href="{{ project.link }}">{{ project.title }}</a>
          {% else %}
            {{ project.title }}
          {% endif %}
        </h3>
        <p>{{ project.description }}</p>
        {% if project.year %}
          <span class="project-year">{{ project.year }}</span>
        {% endif %}
      </div>
    </li>
  {% endfor %}
</ul>
