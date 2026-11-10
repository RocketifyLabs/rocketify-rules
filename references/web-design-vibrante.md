# Web Design Vibrante - Padrões Olfatto

Guia de padrões visuais para redesigns impactantes inspirados em Droper.app.

## Core Principles

- Gradientes vibrantes (pink → purple → blue) para identidade visual forte
- Animações sutis (shimmer, pulse-glow, float) para profundidade
- Cards com hover transform e bordas destacadas
- Admin estilo Color Admin com gradientes e sombras premium

## CSS Tokens Essenciais

```css
:root {
  --vibrant-pink: oklch(70% 0.18 35);
  --vibrant-purple: oklch(68% 0.16 310);
  --vibrant-blue: oklch(65% 0.14 250);
  --gradient-primary: linear-gradient(135deg, var(--vibrant-pink) 0%, var(--vibrant-purple) 70%, var(--vibrant-blue) 100%);
  --gradient-secondary: linear-gradient(135deg, var(--vibrant-teal) 0%, var(--accent-gold) 100%);
  
  /* Radius extras - adicionais ao Tailwind core */
  --radius-sm: 8px;
  --radius-card: 12px;
}
```

## Animações Essenciais

```css
@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}

@keyframes pulse-glow {
  0%, 100% { box-shadow: 0 0 0 0 oklch(70% 0.18 35 / 0.4); }
  50% { box-shadow: 0 0 0 20px oklch(70% 0.18 35 / 0); }
}

@keyframes float {
  0%, 100% { transform: translateY(0) rotate(0deg); }
  50% { transform: translateY(-10px) rotate(1deg); }
}
```

## Componentes a Transformar

### Hero Section
- Background: partículas animadas + gradientes dinâmicos
- Título: shimmer animation sobre gradiente
- Botão primário: pulse-glow + shimmer overlay

### Cards de Ranking
- Hover: translateX(8px) + box-shadow + border-left primária
- Position badge: coral gradient com text-shadow

### Admin Dashboard
- Sidebar: border-left primária + hover destacado
- Cards de overview: shimmer no hover, sombra premium
- Tabela: linha hover colorida com transform translateX(4px)

## Validação

1. Build: `pnpm --filter @olfatto/web build` → exit 0
2. Verificar staging URL
3. Elementos visíveis no DOM para validar animações

## Variáveis de Cor Disponíveis

| Token | Valor | Uso |
|-------|-------|-----|
| `--vibrant-pink` | oklch(70% 0.18 35) | Destaque primário |
| `--vibrant-purple` | oklch(68% 0.16 310) | Gradiente secundário |
| `--vibrant-blue` | oklch(65% 0.14 250) | Gradiente final |
| `--accent-gold` | oklch(85% 0.18 75) | CTA e destaques |
| `--accent-coral` | oklch(75% 0.18 22) | Badges de posição |