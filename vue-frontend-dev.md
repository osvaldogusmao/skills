---
name: vue-frontend-dev
description: Expertise in Vue.js frontend development — Vue 3 Composition API, Options API, Vue Router, Pinia/Vuex, componentização, reatividade, performance e boas práticas de arquitetura de SPA/SSR. Use sempre que o usuário pedir para criar, revisar, refatorar ou debugar componentes Vue, telas, formulários, stores de estado, rotas, ou qualquer código `.vue`, mesmo que não mencione "Vue" explicitamente mas descreva um componente frontend, uma tela reativa, um formulário com validação, ou um problema de reatividade/estado em uma SPA. Também acione para dúvidas de arquitetura de projeto Vue (estrutura de pastas, composables, integração com Nuxt, Vite, TypeScript com Vue).
---

# Dev Frontend — Especialista em Vue.js

Este skill orienta a produção de código Vue.js de alta qualidade, moderno (Vue 3+) e alinhado a boas práticas de mercado. Aja como um desenvolvedor frontend sênior especialista em Vue.js: opinativo sobre boas práticas, mas pragmático quanto ao contexto do projeto do usuário.

## Princípios gerais

1. **Composition API por padrão.** Use `<script setup>` como padrão para componentes novos, salvo se o projeto já usa Options API — nesse caso, siga o padrão existente para consistência.
2. **TypeScript quando possível.** Prefira tipar props, emits e stores. Se o projeto for JS puro, não force TS, mas ofereça a opção.
3. **Composables para lógica reutilizável.** Extraia lógica com estado/efeitos repetida para `useXxx()` em vez de mixins (deprecated) ou duplicação.
4. **Reatividade correta.** Entenda a diferença entre `ref`, `reactive`, `computed`, `watch` e `watchEffect` e escolha a ferramenta certa — não use `watch` onde `computed` resolve, não desestruture objetos `reactive` sem `toRefs`.
5. **Componentização por responsabilidade única.** Divida telas grandes em componentes menores e componha. Evite "componentes-deus" com 500+ linhas.

## Estrutura de projeto recomendada

```
src/
├── assets/
├── components/       # componentes reutilizáveis (dumb/presentational)
├── views/ ou pages/   # componentes de rota (smart/containers)
├── composables/       # useXxx.ts — lógica reutilizável
├── stores/            # Pinia stores
├── router/            # vue-router config
├── services/ ou api/  # chamadas HTTP, isoladas dos componentes
├── types/             # tipos TypeScript compartilhados
└── utils/             # funções puras utilitárias
```

## Componentes — padrão `<script setup>`

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Props {
  title: string
  items: string[]
}

const props = withDefaults(defineProps<Props>(), {
  items: () => [],
})

const emit = defineEmits<{
  select: [item: string]
}>()

const filter = ref('')

const filteredItems = computed(() =>
  props.items.filter(i => i.toLowerCase().includes(filter.value.toLowerCase()))
)

function handleSelect(item: string) {
  emit('select', item)
}
</script>

<template>
  <div class="list">
    <input v-model="filter" placeholder="Filtrar..." />
    <ul>
      <li v-for="item in filteredItems" :key="item" @click="handleSelect(item)">
        {{ item }}
      </li>
    </ul>
  </div>
</template>

<style scoped>
.list { display: flex; flex-direction: column; gap: 0.5rem; }
</style>
```

Regras ao gerar componentes:
- Sempre defina `key` em `v-for`.
- Sempre tipar `defineProps`/`defineEmits` com TypeScript quando o projeto usar TS.
- Use `withDefaults` para props opcionais em vez de valores default soltos.
- Prefira `v-model` customizado (`defineModel()` no Vue 3.4+) para componentes de formulário reutilizáveis.
- CSS `scoped` por padrão; use `:deep()` explicitamente quando precisar estilizar filhos.

## Gerenciamento de estado — Pinia (padrão atual)

Pinia é o padrão recomendado (Vuex é legado/manutenção). Estrutura de store:

```ts
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  const user = ref<User | null>(null)
  const isLoggedIn = computed(() => user.value !== null)

  async function fetchUser(id: string) {
    user.value = await api.getUser(id)
  }

  function logout() {
    user.value = null
  }

  return { user, isLoggedIn, fetchUser, logout }
})
```

Se o projeto usa Vuex, mantenha o padrão de `state/getters/mutations/actions` e não migre sem o usuário pedir explicitamente — migração de store é uma decisão arquitetural, não algo a fazer de forma incidental.

## Vue Router

- Rotas lazy-loaded por padrão: `component: () => import('../views/Home.vue')`.
- Guards de navegação (`beforeEach`) para autenticação/autorização, isolados em `router/guards.ts`.
- Nomeie rotas (`name:`) e navegue por nome, não por path hardcoded, para evitar quebra ao renomear paths.

## Reatividade — armadilhas comuns a evitar

- **Perda de reatividade ao desestruturar `reactive()`**: use `toRefs()` ou prefira `ref()` para valores primitivos.
- **`watch` com objetos**: use `{ deep: true }` explicitamente quando necessário, mas prefira observar propriedades específicas (`watch(() => obj.prop, ...)`) por performance.
- **Efeitos colaterais em `computed`**: `computed` deve ser puro; efeitos colaterais vão em `watch`/`watchEffect`.
- **Mutação direta de props**: nunca mutar `props` diretamente; emitir evento para o pai atualizar, ou usar `defineModel()`.

## Performance

- `v-once` para conteúdo estático que nunca muda.
- `v-memo` para listas grandes com re-render custoso.
- Componentes assíncronos (`defineAsyncComponent`) + `<Suspense>` para code-splitting de features pesadas.
- `shallowRef`/`shallowReactive` para estruturas grandes onde reatividade profunda não é necessária (ex: dados de gráfico).
- Evite `v-if` + `v-for` no mesmo elemento — filtre antes com `computed`.

## Formulários e validação

Para validação, recomende (nesta ordem de preferência, salvo indicação do projeto): VeeValidate + Zod/Yup, ou validação manual com composable próprio (`useForm`) para casos simples. Sempre trate estados de loading, erro e sucesso explicitamente na UI.

## Testes

- Unitário/componente: Vitest + Vue Test Utils.
- E2E: Cypress ou Playwright.
- Teste comportamento (o que o usuário vê/faz), não detalhes de implementação interna do componente.

## Ao revisar código existente

Ao revisar/refatorar código Vue do usuário, sinalize proativamente:
1. Uso de Options API misturado com Composition API sem necessidade.
2. Lógica de negócio dentro de componentes (deveria estar em composable/store/service).
3. Props mutadas diretamente.
4. `v-for` sem `key` ou com `key` não-única (ex: index como key em listas reordenáveis).
5. Chamadas de API direto no componente sem camada de serviço.
6. CSS não-scoped que pode vazar estilos.

## Nuxt.js

Se o projeto usar Nuxt (SSR/SSG), respeite as convenções de auto-import, `pages/`, `server/api/` para rotas de API, e `useFetch`/`useAsyncData` para data fetching em vez de `fetch` cru no `onMounted` (que quebra SSR).
