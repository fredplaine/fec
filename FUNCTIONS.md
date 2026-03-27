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

### `speakPresidentLine(text)` (Lecture au clic pendant que vous jouez)

**États** :
- `idle` → premier clic = commencer à lire
- `playing` → clic = pause (cancel + save chunk index)
- `paused` → clic = reprendre du chunk sauvegardé

**Refs stockées** :
- `presidentChunksRef.current` = chunks du texte
- `presidentChunkIndexRef.current` = index courant
- `presidentTimeoutRef.current` = timeout de pause (important pour limiter)

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

## Variables d'État Essentielles

| Var | Type | Rôle |
|-----|------|------|
| `selectedRole` | string | Rôle du joueur actuel |
| `turboMode` | boolean | Mode accéléré activé |
| `currentSection` | number | Section courante [0, sections.length) |
| `currentLine` | number | Ligne courante dans la section |
| `isPlaying` | boolean | Lecture en cours (démarre haut-parleurs) |
| `presidentSpeechState` | 'idle'\|'playing'\|'paused' | État speech du rôle sélectionné |

---

## Checklist avant modification du Turbo

- [ ] Turbo saute les répliques d'autres rôles → findCueLineIndex() fonctionne
- [ ] Cue line lue partiellement → getLastSentence() retourne 2 phrases
- [ ] isCueLine bien détecté avant speak() ?
- [ ] Cleanup fait partout (handleNext, handlePrevious, section selector)
- [ ] Pas de double-speech (cancel() appelé à temps)
- [ ] Timeouts pas stockés → presidentTimeoutRef.current = null après clearTimeout()

---

## Refs pour Play/Pause Turbo

`speechSynthesis.pause()/resume()` **NE MARCHE PAS** (crash à ponctuation).

**Solution correcte** :
- `cancel()` + rebouclage sur `presidentChunksRef[presidentChunkIndexRef]`
- Sauvegarder l'index dans une ref AVANT l'utterance onend
- Resume = relancer speakPresidentFromChunk() avec cet index
