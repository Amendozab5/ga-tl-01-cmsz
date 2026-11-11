# GA-TL-01 — Grupo CMSZ

## Cooperativa Ahorro Futuro — Escenario 4

**Asignatura:** Aplicaciones Distribuidas (ISR-701) — Sistemas Distribuidos  
**Periodo:** 2026-2027 SPA  
**Docente:** Prof. Guerrero Ulloa G.C.  
**Entrega:** miércoles 11/11/2026, 23:59

---

## Integrantes

| Inicial | Nombre completo | GitHub | Rol |
|---|---|---|---|
| **C** | _CEDEÑO INTRIAGO JILMAR JESUS_ | [@jcedenoi4](https://github.com/jcedenoi4) | Coordinación + transferencias |
| **M** | _MENDOZA BERMELLO ANGELLO AGUSTIN_ | [@Amendozab5](https://github.com/Amendozab5) | Diagrama + cajeros |
| **S** | _SANCHEZ VERA DARWIN JOEL_ | [@DarwinDevCode](https://github.com/DarwinDevCode) | Núcleo + notas |
| **Z** | _ZAMBRANO TRIVIÑO LEONEL LEONIDAS_ | [@lzambranot6-sys](https://github.com/lzambranot6-sys) | Eventos + entrega |

> Cada integrante debe hacer sus propios commits. El historial es la evidencia del aporte individual.

---

## Contenido del repositorio

```text
.
├── README.md
├── .gitignore
├── diagrama/
│   ├── sistema-bancario.drawio
│   ├── sistema-bancario.png
│   └── sistema-bancario.pdf
├── notas/
│   ├── notas-presentacion.tex
│   ├── notas-presentacion.pdf
│   └── figuras/
├── docs/
│   ├── matriz-interacciones.md
│   └── flujos/
│       ├── PLANTILLA.md
│       ├── flujo-1.md
│       ├── flujo-2.md
│       ├── flujo-3.md
│       ├── flujo-4.md
│       ├── flujo-5.md
│       ├── flujo-6.md
│       ├── flujo-7.md
│       └── flujo-8.md
└── entrega/
    └── sga.pdf
```

---

## Diagrama

- **Editable:** `diagrama/sistema-bancario.drawio`
- **Exportaciones:** `diagrama/sistema-bancario.png` (300 DPI) y `diagrama/sistema-bancario.pdf`

### Cómo abrirlo y exportarlo

1. Abrir `diagrama/sistema-bancario.drawio` en [diagrams.net](https://app.diagrams.net/) o en draw.io Desktop.
2. Revisar que estén los 8 flujos numerados, la leyenda, los actores, la base de datos y la red externa.
3. `File > Export as > PNG`:
   - Seleccionar **Page**.
   - **300 DPI**.
   - Fondo no transparente.
   - Guardar como `diagrama/sistema-bancario.png`.
4. `File > Export as > PDF`:
   - Seleccionar **Page**.
   - Guardar como `diagrama/sistema-bancario.pdf`.
5. Guardar el editable con `Ctrl+S`.

---

## Notas de presentación

- **Fuente LaTeX:** `notas/notas-presentacion.tex`
- **PDF compilado:** `notas/notas-presentacion.pdf`

### Cómo compilar

Con TeX Live o MiKTeX:

```bash
cd notas
pdflatex notas-presentacion.tex
pdflatex notas-presentacion.tex
```

O con `latexmk`:

```bash
cd notas
latexmk -pdf notas-presentacion.tex
```

Si usan Overleaf, descarguen el `.tex` y el `.pdf` y súbanlos al repositorio; no dejen solo el enlace.

---

## Entrega en el SGA

En `entrega/sga.pdf` va **solamente** el documento de una página con:

- Datos identificativos del equipo.
- Integrantes.
- Escenario.
- URL completa del repositorio, en una sola línea.

El diagrama y las notas **no** se suben al SGA: van en este repositorio.

---

## Evidencia de aportes

| Integrante | Aporte principal | Archivos / commits |
|---|---|---|
| **C** | Matriz de interacciones, flujos 1-2, F2 | `docs/matriz-interacciones.md` |
| **M** | Diagrama completo, flujos 4-5, F1, export | `diagrama/*`, `docs/flujos/flujo-4.md`, `flujo-5.md` |
| **S** | Flujos 3 y 6, notas LaTeX, R2/R5 | `notas/*`, `docs/flujos/flujo-3.md`, `flujo-6.md` |
| **Z** | Flujos 7-8, README, SGA, QA | `README.md`, `entrega/sga.pdf`, `docs/flujos/flujo-7.md`, `flujo-8.md` |

Verificar autoría:

```bash
git log --oneline --author="Nombre del integrante"
```

---

## Reglas del equipo

- Solo **M** edita el `.drawio` para evitar conflictos.
- Cada dueño escribe la ficha de sus flujos en `docs/flujos/`.
- Antes de editar: `git pull --rebase`.
- Commits pequeños y con mensaje claro.
- No subir archivos temporales, auxiliares de LaTeX ni PDFs de contexto para IA.
- El repositorio debe ser **público** y abrirse en incógnito.

---

## Enlaces

- **Repositorio:** `https://github.com/Amendozab5/ga-tl-01-cmsz`
- **Diagrama:** `diagrama/sistema-bancario.pdf`
- **Notas:** `notas/notas-presentacion.pdf`
