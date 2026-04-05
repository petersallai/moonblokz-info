# MoonBlokz Website Summary

## Availability

The public MoonBlokz website is available at:

- https://moonblokz.com

## Current Scope

The website is a public-facing summary site for MoonBlokz.

Its current structure includes:

1. **Mission / Hero**
   - Presents MoonBlokz as offline-first local DePIN infrastructure for constrained environments.
   - Frames the project as a transaction and coordination layer for places where reliable connectivity does not exist.

2. **Use Cases**
   - Highlights ten representative use-case areas through dedicated public detail pages.
   - The detailed list and short descriptions of those current public-facing use cases are maintained in [`moonblokz-overview.md`](./moonblokz-overview.md) to avoid duplicating the same positioning material across multiple knowledge-base files.

3. **Why MoonBlokz / Why now**
   - Explains why conventional blockchain assumptions break down in infrastructure-constrained environments.
   - Positions MoonBlokz as local coordination infrastructure for environments where operations continue without dependable connectivity.

4. **Technology**
   - Summarizes key technical properties, including:
     - radio-native infrastructure,
     - constrained-device operation,
     - scalability from small deployments to much larger local networks,
     - best-effort synchronization,
     - bounded behavior,
     - field observability.

5. **Series**
   - Links to the MoonBlokz Medium series as the main long-form explanatory material.

6. **Repositories**
   - Presents MoonBlokz as a fully open-source project and links to selected repositories.

7. **Footer / Contact**
   - Links to GitHub, Medium, and the public contact email address.

## Implementation Notes

The current public site uses MoonBlokz branding, a dark technical visual style, animated starfield background treatment, a one-page homepage structure, and separate static pages for each listed use case.

## Deployment Form

The current MoonBlokz website is packaged in a way that supports static deployment.

This means the site can be served from a static web server without requiring a dynamic application backend for normal page delivery.

### Static Deployment Characteristics

The current static deployment package includes:

- a prerendered homepage,
- prerendered HTML entry files for each public use-case detail page,
- bundled JavaScript and CSS assets,
- public image assets,
- use-case artwork,
- favicon and touch-icon assets,
- and a fallback `404.html` file.

### Public Route Scope

The current static deployment covers:

- the homepage,
- and the public use-case detail routes exposed on the website.

### Operational Note

The static deployment package is intended to be copied to a static web server as a publishable website artifact.
