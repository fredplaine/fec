# Théâtre Répétition - Fonctions Clés

## Logique Turbo (Mode Accéléré)

**Comportement à préserver** :
- Mode Turbo saute TOUTES les répliques d'autres rôles
- Seule réplique lue des "autres" : la cue line (juste avant votre rôle)
- Pour la cue line : lire UNIQUEMENT les **2 dernières phrases** (`getLastSentence()`)
- Enchaînement normal pour les répliques de votre rôle

### Flux Turbo dans `speak()` :
```
speak(text, isCueLine) est appelé dans useEffect [currentLine, isPlaying, ...]
  
if (isCueLine && turboMode):
  → textToSpeak = getLastSentence(text)  // 2 dernières phrases
  → after speak finish: setCurrentLine(+1)  // avancer d'1 ligne
  
else if (!isMyRole && turboMode):
  → findCueLineIndex() trouve la ligne juste avant prochaine réplique
  → setTimeout(() => setCurrentLine(cueIdx), 300)  // SAUTER à cue line
  
else:
  → read full text, advance normally
```

**Clé** : `isCueLine` = `turboMode && !!sectionLines[currentLine + 1] && isMyRole(sectionLines[currentLine + 1])`

---

## Fonction `findCueLineIndex(sectionIdx, fromLine)`

**Rôle** : Trouve l'index de la ligne qui **précède** la prochaine réplique du rôle sélectionné.

**Entrée** :
- `sectionIdx` = section courante (ou suivante si on cherche au-delà)
- `fromLine` = ligne de départ (exclu) pour chercher après

**Sortie** :
- `i - 1` où `i` = index de la première réplique du rôle trouvée
- `null` si aucune réplique du rôle trouvée

**Exemple** : 
```
selectedRole = "Le Président"
Ligne 3: "Premier Juré" → pas mon rôle
Ligne 4: "Le Président" → mon rôle trouvé!
findCueLineIndex(..., 0) retourne 3 (la ligne juste avant)
```

---

## Fonction `getLastSentence(text)`

**Rôle** : Extrait les **2 dernières phrases** d'un texte.

**Algorithme** :
1. Nettoyer : supprimer didascalies `[...]`, normaliser espaces
2. Splitter par ponctuation finale `(?<=[.!?])\s+`
3. Si ≤ 2 phrases : retourner tout
4. Sinon : `slice(-2).join(' ')` = 2 dernières phrases

**Indispensable pour Turbo** : lire uniquement ce qui compte avant votre réplique.

---

## Speech Synthesis avec Pauses (Ponctuation)

### `speakWithPunctuationPauses(text, opts, onFinished)`

**Pauses appliquées** :
- `,` ou `;` ou `:` → `commaPause` (défaut 0.25s)
- `.` ou `!` ou `?` → `sentencePause` (défaut 0.75s)

**Important** : Chaque chunk est une utterance séparée → pauses=arrêts naturels

### `speakPresidentLine(text)` (appelée par `handleTextTap` sur simple tap)

**États** :
- `idle` → commence à lire depuis le début
- `playing` → pause (cancel + sauvegarde chunk index)
- `paused` → reprend du chunk sauvegardé

**Refs stockées** :
- `presidentChunksRef.current` = chunks du texte
- `presidentChunkIndexRef.current` = index courant
- `presidentTimeoutRef.current` = timeout de pause
- `presidentSpeechOptsRef.current` = opts voix (pour double-tap)

---

### `handleTextTap(text)` — Séquence complète de taps sur la réplique

**Séquence normale** (hors mode souffleur) :
| Tap | État avant | Action |
|-----|-----------|--------|
| 1er | texte caché | `setRevealedLine(true)` — révèle le texte |
| 2ème | `idle` | `speakPresidentLine` → démarre lecture |
| 3ème | `playing` | Pause (cancel + save index) |
| 4ème | `paused` | Reprend depuis `presidentChunkIndexRef` |

**Double-tap (< 300ms)** : reset au début, texte reste visible, **lecture ne démarre PAS** (état `'paused'` à l'index 0)
```js
const lastTapTimeRef = useRef(0); // millisecondes du dernier tap

const handleTextTap = (text) => {
    const now = Date.now();
    const timeSinceLastTap = now - lastTapTimeRef.current;
    lastTapTimeRef.current = now;

    if (timeSinceLastTap < 300) {
        // Double-tap : cancel + reset index à 0 + état paused (pas de lecture)
        window.speechSynthesis.cancel();
        clearTimeout(presidentTimeoutRef.current);
        presidentTimeoutRef.current = null;
        // Reconstruire chunks si idle sans lecture préalable
        if (!presidentChunksRef.current) { /* rebuild opts + chunks */ }
        presidentChunkIndexRef.current = 0;
        setPresidentSpeechState('paused'); // ← paused, pas playing
        return;
    }
    speakPresidentLine(text); // simple tap
};
```

**Hint tooltip** : `'Cliquer pour entendre · Double-tap pour reprendre depuis le début'`

> Après un double-tap, le prochain tap simple déclenche `speakPresidentLine` en état `'paused'` → reprend depuis le chunk 0.

---

## Cleanup obligatoire partout

Quand on navigue (handleNext, handlePrevious, handleReset, changement de section/rôle) :

```js
setPresidentSpeechState('idle');
if (presidentTimeoutRef.current) {
    clearTimeout(presidentTimeoutRef.current);
    presidentTimeoutRef.current = null;
}
window.speechSynthesis.cancel();
```

**Raison** : éviter les utterances qui se chevauchent, éviter les timeouts qui traînent.

---

---

## Mode Souffleur (`showPrompter`)

Le souffleur s'active via le bouton 👁️. Il ne fonctionne que pour la réplique du rôle sélectionné.

### États (`prompterState`)
```
'hidden' → timer décompte → 'hint' → clic → 'full' → clic → 'audio' → clic → 'full'
```

| État | Comportement affiché |
|------|----------------------|
| `'hidden'` | `...` + compteur "Indice dans Xs" |
| `'hint'` | 2 premiers mots + `...` |
| `'full'` | Texte complet visible |
| `'audio'` | Texte complet + lecture en cours |

### `getPrompterText(text)`
- `'hidden'` ou `!showPrompter` → retourne `'...'`
- `'hint'` → `words.slice(0, 2).join(' ') + ' ...'`
- `'full'` ou `'audio'` → texte complet

### `handlePrompterClick()`
- N'agit que si `showPrompter && isMyRole(currentLineData)`
- `'hidden'` ou `'hint'` → passe à `'full'`
- `'full'` → passe à `'audio'` + appelle `speakPresidentLine(text)`
- `'audio'` → passe à `'full'`

### Timer souffleur
`useEffect` sur `[currentSection, currentLine, showPrompter, prompterState, prompterDelay]` :
- Si `showPrompter && isMyRole && prompterState === 'hidden'` → décompte de `prompterDelay` secondes par intervalle
- Quand compteur atteint 0 → `setPrompterState('hint')`

### Réinitialisation souffleur
`useEffect` sur `[currentSection, currentLine, prompterDelay, selectedRole]` :
- Dans tous les cas : `setPrompterState('hidden')`

### Variables souffleur
| Var | Type | Défaut | Rôle |
|-----|------|--------|------|
| `showPrompter` | boolean | false | Souffleur activé |
| `prompterState` | 'hidden'\|'hint'\|'full'\|'audio' | 'hidden' | Étape d'affichage |
| `prompterDelay` | number | 3 | Secondes avant indice (1-9) |
| `prompterTimer` | number | 3 | Compteur actuel affiché |

---

## Marque-pages

Stockés dans `localStorage` clé `femmesTRAE_bookmarks`.  
Clé de marque-page : `"${currentSection}-${currentLine}"`.

### Fonctions
| Fonction | Rôle |
|----------|------|
| `toggleBookmark()` | Ajoute ou supprime le marque-page courant |
| `isBookmarked()` | Retourne `true` si la ligne courante est marquée |
| `goToBookmark(section, line)` | Navigue + `setIsPlaying(false)`, `setRevealedLine(false)`, ferme la liste |
| `removeBookmark(key)` | Supprime un marque-page par clé |

### Structure d'un marque-page
```js
{
  section: number,
  line: number,
  character: string,
  text: string,    // 50 premiers caractères + '...'
  date: string,    // ISO
  category: 'À travailler'
}
```

---

## Gestes Tactiles (Swipe)

`handleTouchStart` / `handleTouchEnd` attachés sur le `<div>` racine.

**Condition de déclenchement** :
- `|diffX| > 80px` ET `|diffX| > |diffY| × 2` (plus horizontal que vertical)

| Geste | Action |
|-------|--------|
| Swipe gauche | `handleNext()` |
| Swipe droite | `handlePrevious()` |

Après le premier swipe, `showSwipeHint` est mis à `false` et enregistré dans `localStorage`.

---

## Variables d'État Essentielles

| Var | Type | Rôle |
|-----|------|------|
| `selectedRole` | string | Rôle du joueur actuel |
| `turboMode` | boolean | Mode accéléré activé |
| `currentSection` | number | Section courante [0, sections.length) |
| `currentLine` | number | Ligne courante dans la section |
| `isPlaying` | boolean | Lecture en cours (démarre haut-parleurs) |
| `revealedLine` | boolean | Texte du rôle révélé (hors souffleur) |
| `presidentSpeechState` | 'idle'\|'playing'\|'paused' | État speech du rôle sélectionné |
| `bookmarks` | object | Marque-pages `{ "sec-line": {...} }` |
| `showPrompter` | boolean | Mode souffleur actif |
| `prompterState` | 'hidden'\|'hint'\|'full'\|'audio' | Étape du souffleur |

---

## Checklist avant modification du Turbo

- [ ] Turbo saute les répliques d'autres rôles → findCueLineIndex() fonctionne
- [ ] Cue line lue partiellement → getLastSentence() retourne 2 phrases
- [ ] isCueLine bien détecté avant speak() ?
- [ ] Cleanup fait partout (handleNext, handlePrevious, section selector)
- [ ] Pas de double-speech (cancel() appelé à temps)
- [ ] Timeouts pas stockés → presidentTimeoutRef.current = null après clearTimeout()

---

## Fonctionnalités NON implémentées

- Aucune à ce jour.

---

## Refs pour Play/Pause Turbo

`speechSynthesis.pause()/resume()` **NE MARCHE PAS** (crash à ponctuation).

**Solution correcte** :
- `cancel()` + rebouclage sur `presidentChunksRef[presidentChunkIndexRef]`
- Sauvegarder l'index dans une ref AVANT l'utterance onend
- Resume = relancer speakPresidentFromChunk() avec cet index
