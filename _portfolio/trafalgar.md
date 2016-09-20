---
title: "Trafalgar Tours"
excerpt: "Complicated integration with internal systems for a global company."
header:
  image: portfolio/trafalgar-header.png
  teaser: portfolio/trafalgar-thumbnail.png
sidebar:
  - title: "Role"
    image: portfolio/trafalgar-logo.png
    image_alt: "Trafalgar"
    text: "Senior Developer"
  - title: "Responsibilities"
    text: "Delivery of modular Sitecore components. Integration with business systems."
---

This project was a rebuild of an existing Sitecore implementation for [Trafalgar Tours](http://www.trafalgar.com). There were a variety of deep integrations to the client's systems. The large number (and complexity) of the interactions with other business systems meant that we had to maintain a large suite of unit and integration tests. This allowed us to deploy to production multiple times per week without introducing bugs. 

We developed a [Feature Toggling library](https://github.com/aqueduct/Aqueduct.Toggles) to allow us to deploy the site even when features were still under development. 

The site was designed and built to be modular, so we built components to support [dynamic placeholders](https://github.com/aqueduct/Aqueduct.Sitecore.DynamicPlaceholders). This allows content editors to have vast flexibility in how they structure their pages, without having to involve the development team.