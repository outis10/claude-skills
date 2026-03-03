---
name: bootswatch-theme-selector
description: "Add a Bootswatch theme selector to JHipster React projects. Use when the user wants to: add Bootstrap themes, add a theme switcher, integrate Bootswatch, add dark/light theme support, or 'agrega selector de temas', 'agrega temas bootstrap', 'theme selector'. Creates a navbar dropdown with 26 Bootswatch themes, Redux state, localStorage persistence, and dynamic navbar color matching."
---

# Bootswatch Theme Selector for JHipster React

Adds a fully functional Bootstrap theme selector to JHipster React projects using Bootswatch CDN. Includes 26 themes, Redux state management, localStorage persistence, dynamic navbar color adaptation, and accessible dropdown menu.

## Prerequisites

- JHipster project with React frontend
- Redux Toolkit configured (standard JHipster setup)
- Reactstrap available (standard JHipster setup)
- FontAwesome configured (standard JHipster setup)

## Implementation Steps

### Step 1: Create the Redux Theme Reducer

Create file `src/main/webapp/app/shared/reducers/theme.ts`:

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { Storage } from 'react-jhipster';

export type ThemeType =
  | 'default'
  | 'cerulean'
  | 'cosmo'
  | 'cyborg'
  | 'darkly'
  | 'flatly'
  | 'journal'
  | 'litera'
  | 'lumen'
  | 'lux'
  | 'materia'
  | 'minty'
  | 'morph'
  | 'pulse'
  | 'quartz'
  | 'sandstone'
  | 'simplex'
  | 'sketchy'
  | 'slate'
  | 'solar'
  | 'spacelab'
  | 'superhero'
  | 'united'
  | 'vapor'
  | 'yeti'
  | 'zephyr';

export interface ThemeState {
  currentTheme: ThemeType;
}

const storedTheme = Storage.local.get('theme') as ThemeType;
const initialTheme: ThemeType = storedTheme || 'default';

const initialState: ThemeState = {
  currentTheme: initialTheme,
};

// Navbar color mapping for each Bootswatch theme
const navbarColors: Record<ThemeType, { bg: string; text: string }> = {
  default: { bg: '#353d47', text: '#fff' },
  cerulean: { bg: '#007bff', text: '#fff' },
  cosmo: { bg: '#2c3e50', text: '#fff' },
  cyborg: { bg: '#222222', text: '#fff' },
  darkly: { bg: '#222222', text: '#fff' },
  flatly: { bg: '#2c3e50', text: '#fff' },
  journal: { bg: '#f8f9f9', text: '#333' },
  litera: { bg: '#f8f9f9', text: '#333' },
  lumen: { bg: '#f8f9f9', text: '#333' },
  lux: { bg: '#fff', text: '#333' },
  materia: { bg: '#e74c3c', text: '#fff' },
  minty: { bg: '#01847f', text: '#fff' },
  morph: { bg: '#1e1e2e', text: '#fff' },
  pulse: { bg: '#f0f0f0', text: '#333' },
  quartz: { bg: '#422d49', text: '#fff' },
  sandstone: { bg: '#d0ccc8', text: '#333' },
  simplex: { bg: '#fff', text: '#333' },
  sketchy: { bg: '#fff', text: '#333' },
  slate: { bg: '#3e4348', text: '#fff' },
  solar: { bg: '#002b36', text: '#fff' },
  spacelab: { bg: '#003366', text: '#fff' },
  superhero: { bg: '#1f1f1f', text: '#fff' },
  united: { bg: '#e74c3c', text: '#fff' },
  vapor: { bg: '#494d5c', text: '#fff' },
  yeti: { bg: '#222', text: '#fff' },
  zephyr: { bg: '#f3f3f1', text: '#333' },
};

// Helper function to load theme from Bootswatch CDN
const loadTheme = (theme: ThemeType) => {
  if (typeof document !== 'undefined') {
    let themeLink = document.getElementById('theme-link') as HTMLLinkElement;

    if (theme === 'default') {
      if (themeLink) {
        themeLink.remove();
      }
    } else {
      if (!themeLink) {
        themeLink = document.createElement('link');
        themeLink.id = 'theme-link';
        themeLink.rel = 'stylesheet';
        document.head.appendChild(themeLink);
      }
      themeLink.href = `https://cdn.jsdelivr.net/npm/bootswatch@5.3.0/dist/${theme}/bootstrap.min.css`;
    }

    // Inject dynamic navbar styles
    const colors = navbarColors[theme];
    let styleElement = document.getElementById('theme-navbar-style') as HTMLStyleElement;

    if (!styleElement) {
      styleElement = document.createElement('style');
      styleElement.id = 'theme-navbar-style';
      document.head.appendChild(styleElement);
    }

    styleElement.textContent = `
      .jh-navbar {
        background-color: ${colors.bg} !important;
      }
      .jh-navbar a,
      .jh-navbar .nav-link,
      .jh-navbar .navbar-brand,
      .jh-navbar .dropdown-toggle {
        color: ${colors.text} !important;
      }
      .jh-navbar .dropdown-item {
        color: #333 !important;
      }
      .jh-navbar .dropdown-item.active,
      .jh-navbar .dropdown-item.active:focus,
      .jh-navbar .dropdown-item.active:hover {
        background-color: #e9ecef !important;
        color: #333 !important;
      }
      .jh-navbar .dropdown-item:hover {
        background-color: #f8f9f9 !important;
      }
      .jh-navbar .navbar-toggler {
        border-color: ${colors.text};
      }
      .jh-navbar .navbar-toggler-icon {
        background-image: url("data:image/svg+xml,%3csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 30 30'%3e%3cpath stroke='${encodeURIComponent(colors.text)}' stroke-linecap='round' stroke-miterlimit='10' stroke-width='2' d='M4 7h22M4 15h22M4 23h22'/%3e%3c/svg%3e");
      }
      .brand-title {
        color: ${colors.text} !important;
      }
      .brand-title:hover {
        color: ${colors.text} !important;
      }
    `;
  }
};

// Load initial theme on module load
loadTheme(initialTheme);

export const ThemeSlice = createSlice({
  name: 'theme',
  initialState,
  reducers: {
    setTheme(state, action: PayloadAction<ThemeType>) {
      const theme = action.payload;
      state.currentTheme = theme;
      Storage.local.set('theme', theme);
      loadTheme(theme);
    },
  },
});

export const { setTheme } = ThemeSlice.actions;

export default ThemeSlice.reducer;
```

### Step 2: Create the ThemeMenu Component

Create file `src/main/webapp/app/shared/layout/menus/theme.tsx`:

```tsx
import React, { useState } from 'react';
import { DropdownItem, Dropdown, DropdownToggle, DropdownMenu } from 'reactstrap';
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import { ThemeType } from 'app/shared/reducers/theme';

export interface IThemeMenuProps {
  currentTheme: ThemeType;
  onClick: (event: any) => void;
}

export const themes: Record<ThemeType, { name: string }> = {
  default: { name: 'Default' },
  cerulean: { name: 'Cerulean' },
  cosmo: { name: 'Cosmo' },
  cyborg: { name: 'Cyborg' },
  darkly: { name: 'Darkly' },
  flatly: { name: 'Flatly' },
  journal: { name: 'Journal' },
  litera: { name: 'Litera' },
  lumen: { name: 'Lumen' },
  lux: { name: 'Lux' },
  materia: { name: 'Materia' },
  minty: { name: 'Minty' },
  morph: { name: 'Morph' },
  pulse: { name: 'Pulse' },
  quartz: { name: 'Quartz' },
  sandstone: { name: 'Sandstone' },
  simplex: { name: 'Simplex' },
  sketchy: { name: 'Sketchy' },
  slate: { name: 'Slate' },
  solar: { name: 'Solar' },
  spacelab: { name: 'Spacelab' },
  superhero: { name: 'Superhero' },
  united: { name: 'United' },
  vapor: { name: 'Vapor' },
  yeti: { name: 'Yeti' },
  zephyr: { name: 'Zephyr' },
};

export const ThemeMenu = ({ currentTheme, onClick }: IThemeMenuProps) => {
  const [dropdownOpen, setDropdownOpen] = useState(false);

  const toggleDropdown = () => setDropdownOpen(!dropdownOpen);

  const handleThemeClick = (event: any) => {
    setDropdownOpen(false);
    // Defer click handler to avoid aria-hidden focus conflict
    Promise.resolve().then(() => {
      onClick(event);
    });
  };

  return (
    <Dropdown nav inNavbar isOpen={dropdownOpen} toggle={toggleDropdown} id="theme-menu">
      <DropdownToggle nav caret className="d-flex align-items-center">
        <FontAwesomeIcon icon="paint-brush" />
        <span>{currentTheme ? themes[currentTheme].name : 'Theme'}</span>
      </DropdownToggle>
      <DropdownMenu end role="menu">
        {(Object.keys(themes) as ThemeType[]).map(theme => (
          <DropdownItem key={theme} value={theme} onClick={handleThemeClick} active={theme === currentTheme} role="menuitem">
            {themes[theme].name}
          </DropdownItem>
        ))}
      </DropdownMenu>
    </Dropdown>
  );
};
```

### Step 3: Register the Theme Reducer

Edit `src/main/webapp/app/shared/reducers/index.ts`:

Add the import (before the jhipster-needle comment):
```typescript
import theme from './theme';
```

Add `theme` to the `rootReducer` object (before the jhipster-needle comment):
```typescript
const rootReducer: ReducersMapObject = {
  authentication,
  locale,
  theme,  // <-- add this line
  // ... rest of reducers
};
```

### Step 4: Export ThemeMenu from Menus Index

Edit `src/main/webapp/app/shared/layout/menus/index.ts`:

Add the export:
```typescript
export * from './theme';
```

### Step 5: Register the FontAwesome Icon

Edit `src/main/webapp/app/config/icon-loader.ts`:

Add the import:
```typescript
import { faPaintBrush } from '@fortawesome/free-solid-svg-icons';
```

Add to `library.add(...)`:
```typescript
faPaintBrush,
```

### Step 6: Integrate ThemeMenu in the Header

Edit `src/main/webapp/app/shared/layout/header/header.tsx`:

Add imports:
```typescript
import { setTheme } from 'app/shared/reducers/theme';
import { ThemeMenu } from '../menus';
```

Inside the Header component, add the theme state and handler:
```typescript
const currentTheme = useAppSelector(state => state.theme.currentTheme);

const handleThemeChange = event => {
  const theme = event.target.value;
  dispatch(setTheme(theme));
};
```

Add the ThemeMenu component in the Nav (before LocaleMenu or AccountMenu):
```tsx
<ThemeMenu currentTheme={currentTheme} onClick={handleThemeChange} />
```

### Step 7: Update Content Security Policy

Edit `src/main/resources/config/application.yml`:

Update the `content-security-policy` under `jhipster.security` to include:
- `style-src`: add `https://cdn.jsdelivr.net` and `https://fonts.googleapis.com`
- `font-src`: add `https://fonts.gstatic.com`

Example:
```yaml
content-security-policy: "default-src 'self'; frame-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://storage.googleapis.com; style-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net https://fonts.googleapis.com; img-src 'self' data:; font-src 'self' data: https://fonts.gstatic.com"
```

## Available Themes (26 Bootswatch Themes)

| Theme | Style |
|-------|-------|
| Default | Standard Bootstrap |
| Cerulean | Clean, blue-toned |
| Cosmo | Modern, minimalist |
| Cyborg | Dark, high-contrast |
| Darkly | Dark mode |
| Flatly | Flat design |
| Journal | Newspaper-like |
| Litera | Light, readable |
| Lumen | Light, shadow-based |
| Lux | Elegant serif |
| Materia | Material Design inspired |
| Minty | Fresh, green-toned |
| Morph | Neumorphism/dark |
| Pulse | Vibrant, purple |
| Quartz | Dark gradient |
| Sandstone | Warm, earthy |
| Simplex | Clean, minimal |
| Sketchy | Hand-drawn |
| Slate | Dark, professional |
| Solar | Solarized palette |
| Spacelab | NASA-inspired |
| Superhero | Dark, heroic |
| United | Ubuntu-inspired |
| Vapor | Retro/vaporwave |
| Yeti | Clean, modern |
| Zephyr | Light, airy |

## Technical Notes

- Themes load dynamically from Bootswatch CDN (`cdn.jsdelivr.net`) without page reload
- Theme selection persists in `localStorage` across sessions
- Navbar colors adapt dynamically to match each theme using injected CSS
- Uses a controlled Dropdown (not UncontrolledDropdown) to avoid aria-hidden focus conflicts
- The `Promise.resolve().then()` pattern defers the click handler to avoid accessibility race conditions
- Bootswatch version pinned to `5.3.0` (compatible with Bootstrap 5.x in JHipster)

## Integration with Other Skills

- This skill is a **frontend-only** feature, no backend changes needed except CSP
- Works alongside **jhipster-react-frontend** patterns for component creation
- Compatible with **kalitron-suite** branding
