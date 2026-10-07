# Actividad guiada: la pantalla FlatList

Vas a construir la pantalla **FlatList** de esta app: una lista de personajes de Rick and Morty que carga de a una página, pinta solo lo que está cerca de la pantalla y se puede refrescar arrastrando hacia abajo.

Al terminar, en el banner de arriba vas a ver que **Datos cargados** crece al hacer scroll y **Tarjetas montadas** se mantiene bajo.

## Antes de empezar

1. Instalá dependencias y levantá la app:

   ```bash
   npm install
   npx expo start
   ```

2. Abrí la app, entrá a **ScrollView** y probá el chip **Todos**. La carga se pone lenta porque se montan todas las tarjetas a la vez. Esa pantalla ya está hecha: no la modifiques. Es el contraste de esta actividad.

3. La ruta `/flatlist` ya existe (`src/app/flatlist.tsx`) y el botón de inicio ya navega hasta ahí. Trabajás solo en ese archivo.

4. No reescribas la API ni los componentes. Ya están listos:

   | Pieza | Dónde | Para qué la usás |
   | --- | --- | --- |
   | `getCharactersPage(page)` | `src/api/rickAndMorty.ts` | Trae 20 personajes y dice si hay más páginas |
   | `Character` | el mismo archivo | Tipo de cada personaje |
   | `CharacterCard` | `src/components/CharacterCard.tsx` | Tarjeta de un personaje. Suma 1 al montarse |
   | `StatsBanner` | `src/components/StatsBanner.tsx` | Muestra datos cargados vs tarjetas montadas |
   | `ScrollTopButton` | `src/components/ScrollTopButton.tsx` | Botón flotante para volver arriba |
   | `colors` | `src/theme/colors.ts` | Paleta. FlatList usa `colors.mint` |

`getCharactersPage` devuelve `{ results, hasMore }`. Si la API responde 429, la función ya reintenta sola.

Reemplazá el contenido de `src/app/flatlist.tsx` por el esqueleto y andá sumando cada paso sobre el anterior.

```tsx
import { StyleSheet, View } from 'react-native';
import { colors } from '../theme/colors';

const FlatListScreen = () => {
  return <View style={styles.screen} />;
};

export default FlatListScreen;

const styles = StyleSheet.create({
  screen: {
    flex: 1,
    backgroundColor: colors.background,
  },
});
```

---

## Paso 1. Una lista con datos fijos

Antes de tocar la red, hacé que `FlatList` pinte tres personajes inventados. Así separás “cómo se renderiza una lista” de “cómo se cargan los datos”.

`FlatList` no recibe hijos como `ScrollView`. Recibe un arreglo en `data` y una función `renderItem`. `keyExtractor` es el equivalente al `key` de un `.map()`.

```tsx
import { useState } from 'react';
import { FlatList, StyleSheet, Text, View } from 'react-native';
import type { Character } from '../api/rickAndMorty';
import { colors } from '../theme/colors';

const MOCK: Character[] = [
  {
    id: 1,
    name: 'Rick Sanchez',
    status: 'Alive',
    species: 'Human',
    image: 'https://rickandmortyapi.com/api/character/avatar/1.jpeg',
    location: { name: 'Citadel of Ricks', url: '' },
  },
  {
    id: 2,
    name: 'Morty Smith',
    status: 'Alive',
    species: 'Human',
    image: 'https://rickandmortyapi.com/api/character/avatar/2.jpeg',
    location: { name: 'Earth', url: '' },
  },
  {
    id: 3,
    name: 'Summer Smith',
    status: 'Alive',
    species: 'Human',
    image: 'https://rickandmortyapi.com/api/character/avatar/3.jpeg',
    location: { name: 'Earth', url: '' },
  },
];

const FlatListScreen = () => {
  const [characters] = useState(MOCK);

  return (
    <View style={styles.screen}>
      <FlatList
        data={characters}
        keyExtractor={(item) => item.id.toString()}
        renderItem={({ item }) => (
          <Text style={styles.row}>
            #{item.id} · {item.name}
          </Text>
        )}
        contentContainerStyle={styles.list}
      />
    </View>
  );
};

export default FlatListScreen;

const styles = StyleSheet.create({
  screen: {
    flex: 1,
    backgroundColor: colors.background,
  },
  list: {
    padding: 16,
  },
  row: {
    fontSize: 16,
    color: colors.text,
    paddingVertical: 12,
  },
});
```

**Comprobá:** en FlatList ves tres nombres. Si falta `keyExtractor`, la consola avisa que cada ítem necesita una key.

---

## Paso 2. La tarjeta real

Cambiá el `Text` por `CharacterCard`. La tarjeta ya trae imagen, especie, ubicación y el color del estado.

```tsx
import CharacterCard from '../components/CharacterCard';
```

Y en la lista:

```tsx
renderItem={({ item }) => <CharacterCard character={item} />}
```

Podés borrar el estilo `row`.

**Comprobá:** las tres tarjetas se ven como en ScrollView. Todavía no hay red: los datos siguen siendo `MOCK`.

---

## Paso 3. La primera página de la API

Sacá el mock. Al entrar a la pantalla pedí la página 1 y guardá el resultado.

Estados que vas a necesitar de acá en adelante:

| Estado | Valor inicial | Qué representa |
| --- | --- | --- |
| `characters` | `[]` | Lo que ya se mostró |
| `page` | `1` | La próxima página a pedir |
| `hasMore` | `true` | Si la API todavía tiene páginas |
| `loading` | `false` | Si hay una petición en curso |
| `error` | `null` | El mensaje si la petición falló |

```tsx
import { useCallback, useEffect, useState } from 'react';
import { FlatList, StyleSheet, View } from 'react-native';
import { getCharactersPage, type Character } from '../api/rickAndMorty';
import CharacterCard from '../components/CharacterCard';
import { colors } from '../theme/colors';

const FlatListScreen = () => {
  const [characters, setCharacters] = useState<Character[]>([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const loadPage = useCallback(async (pageToLoad: number, replace = false) => {
    setLoading(true);
    setError(null);

    try {
      const { results, hasMore: more } = await getCharactersPage(pageToLoad);
      setCharacters((prev) => (replace ? results : [...prev, ...results]));
      setPage(pageToLoad + 1);
      setHasMore(more);
    } catch (e: unknown) {
      setError(e instanceof Error ? e.message : 'Error desconocido');
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    loadPage(1, true);
  }, [loadPage]);

  return (
    <View style={styles.screen}>
      <FlatList
        data={characters}
        keyExtractor={(item) => item.id.toString()}
        renderItem={({ item }) => <CharacterCard character={item} />}
        contentContainerStyle={styles.list}
      />
    </View>
  );
};
```

`replace` todavía no cambia nada visible: la lista arranca vacía. Lo vas a usar en el pull to refresh.

Fijate en la actualización de `characters`:

```tsx
setCharacters((prev) => (replace ? results : [...prev, ...results]));
```

Usá la forma con función (`prev => …`). Si leés `characters` directo desde el closure, una segunda página puede pisar a la primera.

**Comprobá:** al entrar aparecen unos 20 personajes. `page` queda en `2`, pero bajar hasta el final no carga más. Eso es el paso que sigue.

---

## Paso 4. Scroll infinito

`onEndReached` se dispara cuando el usuario se acerca al final. `onEndReachedThreshold={0.5}` significa “cuando falta medio alto de pantalla”.

Agregá el handler y conectalo a la lista:

```tsx
const handleEndReached = () => {
  if (hasMore && !loading && !error) loadPage(page);
};
```

```tsx
onEndReached={handleEndReached}
onEndReachedThreshold={0.5}
```

**Comprobá:** al acercarte al final, la lista crece de a 20. Si la consola o la red muestran la misma página dos veces, seguí al paso 5 antes de buscar otro bug: `onEndReached` puede dispararse varias veces mientras `loading` todavía es `false`.

---

## Paso 5. Una sola petición a la vez

`setLoading(true)` no alcanza. React aplica el estado después del render, y `onEndReached` puede llamarse otra vez en el medio.

Guardá un ref que cambia al instante:

```tsx
import { useCallback, useEffect, useRef, useState } from 'react';

const loadingRef = useRef(false);

const loadPage = useCallback(async (pageToLoad: number, replace = false) => {
  if (loadingRef.current) return;
  loadingRef.current = true;
  setLoading(true);
  setError(null);

  try {
    const { results, hasMore: more } = await getCharactersPage(pageToLoad);
    setCharacters((prev) => (replace ? results : [...prev, ...results]));
    setPage(pageToLoad + 1);
    setHasMore(more);
  } catch (e: unknown) {
    setError(e instanceof Error ? e.message : 'Error desconocido');
  } finally {
    loadingRef.current = false;
    setLoading(false);
  }
}, []);
```

Y en el handler, el ref ya cubre el caso de “está cargando”:

```tsx
const handleEndReached = () => {
  if (hasMore && !error) loadPage(page);
};
```

**Comprobá:** cada página nueva trae ids que siguen a los anteriores (`1…20`, después `21…40`). No se repite el bloque.

---

## Paso 6. Header, separador, vacío y pie

`FlatList` tiene lugares fijos para lo que no es un ítem. No los metas dentro de `renderItem`: el header y el footer se pintan una vez, y el separador va entre ítems.

Creá estos dos componentes **debajo** de `FlatListScreen` (o arriba, da igual: no se llaman mientras el módulo se está cargando).

```tsx
import { ActivityIndicator, FlatList, StyleSheet, Text, View } from 'react-native';

const Separator = () => <View style={styles.separator} />;

interface FooterProps {
  loading: boolean;
  error: string | null;
  hasMore: boolean;
  onRetry: () => void;
}

const Footer = ({ loading, error, hasMore, onRetry }: FooterProps) => {
  if (loading) {
    return <ActivityIndicator style={styles.footer} color={colors.mintStrong} />;
  }
  if (error) {
    return (
      <Text style={[styles.message, styles.footer]} onPress={onRetry}>
        {error}. Toca para reintentar.
      </Text>
    );
  }
  if (!hasMore) {
    return <Text style={[styles.message, styles.footer]}>Llegaste al final</Text>;
  }
  return null;
};
```

Conectalos en el `FlatList`:

```tsx
ListHeaderComponent={
  <Text style={styles.hint}>
    FlatList solo monta las tarjetas cercanas a la pantalla. Baja hasta el
    final: los datos cargados crecen, pero las tarjetas montadas no.
  </Text>
}
ItemSeparatorComponent={Separator}
ListEmptyComponent={
  loading ? null : <Text style={styles.message}>No hay personajes para mostrar</Text>
}
ListFooterComponent={
  <Footer
    loading={loading}
    error={error}
    hasMore={hasMore}
    onRetry={() => loadPage(page)}
  />
}
```

Estilos que faltan:

```tsx
list: {
  padding: 16,
  paddingBottom: 100,
},
hint: {
  fontSize: 13,
  color: colors.textSoft,
  lineHeight: 18,
  marginBottom: 12,
},
separator: {
  height: 12,
},
footer: {
  marginVertical: 20,
},
message: {
  fontSize: 15,
  color: colors.textSoft,
  textAlign: 'center',
},
```

**Comprobá:**

- Hay un texto de ayuda arriba de la primera tarjeta.
- Entre tarjetas hay un espacio de 12 px, sin envolver cada tarjeta en un `View` extra.
- Mientras carga la página siguiente, el pie muestra el spinner.
- Si cortás la red, el pie muestra el error y al tocarlo reintenta.

---

## Paso 7. Pull to refresh

Arrastrar hacia abajo estando arriba tiene que volver a pedir la página 1 y reemplazar la lista, no concatenarla.

```tsx
const [refreshing, setRefreshing] = useState(false);

const handleRefresh = async () => {
  setRefreshing(true);
  await loadPage(1, true);
  setRefreshing(false);
};
```

```tsx
refreshing={refreshing}
onRefresh={handleRefresh}
```

El spinner del pie no debería aparecer al mismo tiempo que el del refresh. Pasale al footer la carga que no es un refresh:

```tsx
loading={loading && !refreshing}
```

**Comprobá:** con la lista scrolleada, volvé arriba, arrastrá hacia abajo y soltá. La lista vuelve a empezar en el personaje 1. `refreshing` es el spinner nativo de arriba; el `ActivityIndicator` del pie es el de las páginas siguientes.

---

## Paso 8. El banner y la virtualización

Importá el banner y ponelo **fuera** del `FlatList`, como hermano de arriba. Si lo metés en la lista, scrollea junto con los ítems y cuesta comparar los dos números.

```tsx
import StatsBanner from '../components/StatsBanner';

<View style={styles.screen}>
  <StatsBanner loaded={characters.length} color={colors.mint} />
  <FlatList /* ... */ />
</View>
```

Después ajustá cuánto mantiene la lista en memoria:

```tsx
initialNumToRender={10}
maxToRenderPerBatch={10}
windowSize={5}
```

| Prop | Qué hace |
| --- | --- |
| `initialNumToRender` | Cuántos ítems pinta en el primer render |
| `maxToRenderPerBatch` | Cuántos agrega en cada tanda al scrollear |
| `windowSize` | Cuántas “pantallas” de alto conserva montadas. El default es 21 |

**Comprobá:** bajá varias páginas.

- **Datos cargados** sube de 20 en 20.
- **Tarjetas montadas** se queda en un número chico (cerca de lo que entra en pantalla, más un margen).
- Entrá a ScrollView con **Todos** y compará: ahí las tarjetas montadas coinciden con los datos cargados.

Esa diferencia es la virtualización. `CharacterCard` está envuelta en `memo` y registra cada montaje en `mountCounter`. Si el contador no se mueve, la tarjeta no se está desmontando: revisá que estés en FlatList y no en ScrollView.

---

## Paso 9. Volver arriba

`ScrollView` scrollea con `scrollTo`. `FlatList` scrollea con `scrollToOffset`, y hace falta un ref para llamar a ese método desde el botón.

```tsx
import { useCallback, useEffect, useRef, useState } from 'react';
import { FlatList } from 'react-native';
import ScrollTopButton from '../components/ScrollTopButton';

const listRef = useRef<FlatList<Character>>(null);
```

```tsx
<FlatList
  ref={listRef}
  /* el resto de las props */
/>

<ScrollTopButton
  color={colors.mint}
  onPress={() => listRef.current?.scrollToOffset({ offset: 0, animated: true })}
/>
```

El botón va en el `View` de la pantalla, no dentro de la lista. Su estilo ya es `position: 'absolute'`.

**Comprobá:** scrolleá, tocá **↑ Arriba** y la lista vuelve al inicio con animación. El banner no se mueve.

---

## Paso 10. `renderItem` estable

Cada render de la pantalla crea una función nueva si `renderItem` está inline. Con `memo`, la tarjeta igual se saltea cuando sus props no cambian, pero la función nueva obliga a `FlatList` a reconciliar la lista.

Dejá una sola función y memorizala:

```tsx
import { FlatList, type ListRenderItem } from 'react-native';

const renderItem: ListRenderItem<Character> = useCallback(
  ({ item }) => <CharacterCard character={item} />,
  []
);
```

```tsx
renderItem={renderItem}
```

El arreglo de dependencias va vacío porque `CharacterCard` no cambia.

---

## Lista de cierre

- [ ] La pantalla abre en `/flatlist` desde el inicio.
- [ ] La primera carga trae 20 personajes, sin mock.
- [ ] Al acercarse al final se concatena la página siguiente, sin duplicar ids.
- [ ] Una página no se pide dos veces seguidas (`loadingRef`).
- [ ] El pie muestra spinner, error con reintento, o el fin de la lista.
- [ ] El pull to refresh reemplaza la lista con la página 1.
- [ ] El banner queda fijo: los datos crecen y las tarjetas montadas no.
- [ ] **↑ Arriba** llama a `scrollToOffset`.
- [ ] `keyExtractor` usa `item.id`.
- [ ] `npx tsc --noEmit` termina sin errores.

## Errores frecuentes

**La misma página entra dos veces.** Falta el `return` temprano con `loadingRef`, o `loadPage` lee `characters` del render anterior en vez de usar `setCharacters(prev => …)`.

**`onEndReached` dispara la página 2 apenas abre la pantalla.** La lista todavía es corta y ya “llegó al final”. Esperá a tener la página 1 y no llames a `loadPage` si `loadingRef.current` es `true`. Con el guard del paso 5 alcanza.

**El banner scrollea.** Está dentro de `ListHeaderComponent`. Tiene que ser hermano del `FlatList`.

**Pull to refresh duplica personajes.** `handleRefresh` tiene que llamar `loadPage(1, true)`. Con `replace` en `false` se concatenan los primeros 20 otra vez.

**El contador de tarjetas montadas iguala al de datos.** Estás renderizando con `.map()` dentro de un `ScrollView`, o `windowSize` quedó tan alto que nunca desmonta. Volvé a `windowSize={5}` y confirmá que el ítem es `CharacterCard`.

**TypeScript se queja del ref.** El tipo es `useRef<FlatList<Character>>(null)`, con el genérico del ítem.
