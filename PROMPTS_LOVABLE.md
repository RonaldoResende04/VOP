# VOP — Prompts para o Lovable

Planner colaborativo (estilo ClickUp/Notion/Trello) para 2–5 pessoas.

**Como usar:** cole um prompt por vez no Lovable, na ordem. Ao fim de cada etapa, teste
(de preferência com a outra pessoa, em outro computador) antes de seguir. Se algo quebrar,
peça a correção ali mesmo antes de avançar — empilhar etapas sobre um erro custa mais créditos.

> Dica: se o Lovable "esquecer" o contexto, comece o prompt com
> "Lembre: o app é o VOP, um planner colaborativo para uma equipe pequena e fechada."

---

## Etapa 0 — Criar o projeto (prompt inicial)

```text
Crie um app web chamado VOP: um planner colaborativo para uma equipe pequena e fechada
(2 a 5 pessoas), no estilo ClickUp/Notion. Interface em português do Brasil.

Stack: React + TypeScript + Tailwind + shadcn/ui. Use o Lovable Cloud como backend
(banco de dados, autenticação e realtime).

Nesta primeira etapa, faça APENAS:
1. Autenticação com email e senha (telas de login, cadastro e "esqueci minha senha").
2. Acesso restrito: crie a tabela `allowed_emails (email text primary key)` e impeça que
   qualquer email fora dessa lista crie conta (valide no backend, não só no front).
   Já insira os emails: SEU_EMAIL@exemplo.com, EMAIL_DA_OUTRA_PESSOA@exemplo.com
3. Tabela `profiles` (id = auth.users.id, full_name, avatar_url, created_at), criada
   automaticamente por trigger quando o usuário se cadastra.
4. Uma função SQL `is_member()` que retorna true se o usuário logado tem email em
   `allowed_emails`. Toda política de RLS do app vai usar essa função.
5. Habilite RLS em todas as tabelas. Usuários não logados não acessam nada.
6. Após o login, mostre uma tela vazia com uma barra lateral (logo "VOP", item
   "Tarefas", item "Notas", e no rodapé o avatar/nome do usuário com opção de sair).

Não crie ainda as telas de tarefas nem de notas.
```

> Troque os emails antes de enviar. Para adicionar alguém depois:
> "Adicione o email X à tabela allowed_emails".

---

## Etapa 1 — Modelo de dados das tarefas + visualização em Lista

```text
Agora crie o modelo de dados de tarefas, com RLS usando is_member() para
select/insert/update/delete em todas as tabelas novas:

- projects: id uuid, name text, color text, icon text (emoji), position int,
  created_by uuid -> profiles, created_at, archived boolean default false
- tasks: id uuid, project_id uuid -> projects (on delete cascade), title text not null,
  description text, status text not null default 'a_fazer'
  (valores: 'a_fazer', 'em_andamento', 'em_revisao', 'concluido'),
  priority text not null default 'media' (valores: 'urgente', 'alta', 'media', 'baixa'),
  assignee_id uuid -> profiles (nullable), start_date date, due_date date,
  position double precision (para ordenação manual), parent_task_id uuid -> tasks
  (subtarefas, nullable), created_by, created_at, updated_at (atualizado por trigger),
  completed_at timestamptz (preenchido por trigger quando status vira 'concluido').
  Garanta que due_date >= start_date quando ambos existirem.

Interface:
- Na barra lateral, em "Tarefas", liste os projetos com botão "+ Novo projeto"
  (nome, emoji e cor). Também um item "Todas as tarefas" no topo.
- Ao abrir um projeto, mostre um cabeçalho com o nome do projeto e um seletor de
  visualização em abas: Lista | Tabela | Kanban | Cronograma (só Lista funciona agora;
  as outras mostram "em breve").
- Visualização LISTA: tarefas agrupadas por status (grupos recolhíveis), cada linha com
  checkbox de concluir, título, badge de prioridade colorido, avatar do responsável e
  data de entrega (vermelha se atrasada). Botão "+ Adicionar tarefa" em cada grupo
  (criação rápida só com título, pressionando Enter).
- Clicar numa tarefa abre um painel lateral (sheet) para editar todos os campos,
  incluindo subtarefas e descrição. Salvar automaticamente ao editar.
- Excluir tarefa com confirmação.

Centralize o estado/consultas de tarefas num hook reutilizável (ex.: useTasks com
TanStack Query), pois as próximas visualizações vão usar os mesmos dados.
```

---

## Etapa 2 — Filtros

```text
Adicione uma barra de filtros acima das visualizações de tarefas, compartilhada por
todas as visualizações (Lista, Tabela, Kanban, Cronograma):

- Filtro por Status (multisseleção)
- Filtro por Prioridade (multisseleção)
- Filtro por Responsável (multisseleção, incluindo opção "Sem responsável" e um atalho
  "Minhas tarefas")
- Campo de busca por texto no título
- Opção "Ocultar concluídas"
- Botão "Limpar filtros" e um indicador com a quantidade de filtros ativos

Os filtros devem ser mantidos ao trocar de visualização e salvos na URL (query string),
para que eu possa mandar o link já filtrado para outra pessoa.
```

---

## Etapa 3 — Kanban

```text
Implemente a visualização KANBAN usando @dnd-kit:

- Uma coluna por status (A fazer, Em andamento, Em revisão, Concluído), com contador
  de tarefas no cabeçalho.
- Cards com título, badge de prioridade, avatar do responsável, data de entrega e
  quantidade de subtarefas concluídas/total.
- Arrastar card entre colunas altera o status; arrastar dentro da coluna altera a
  ordem (campo position). Atualização otimista (move na hora e salva em seguida;
  se falhar, desfaz e mostra um toast de erro).
- "+ Adicionar" no rodapé de cada coluna.
- Clicar no card abre o mesmo painel lateral de edição da Lista.
- Respeitar os filtros ativos.
```

---

## Etapa 4 — Tabela

```text
Implemente a visualização TABELA (estilo planilha), usando TanStack Table:

- Colunas: Título, Status, Prioridade, Responsável, Início, Entrega, Projeto
  (só em "Todas as tarefas"), Criado em.
- Edição direta na célula: status/prioridade/responsável por select, datas por
  date picker, título por input.
- Ordenar clicando no cabeçalho; mostrar/ocultar colunas; redimensionar colunas.
- Seleção múltipla de linhas com ações em massa: mudar status, mudar responsável,
  mudar prioridade, excluir.
- Respeitar os filtros ativos.
```

---

## Etapa 5 — Cronograma (Gantt)

```text
Implemente a visualização CRONOGRAMA (Gantt):

- Linha do tempo horizontal com zoom Dia / Semana / Mês e botão "Hoje".
- Cada tarefa com start_date e due_date vira uma barra colorida pela prioridade (ou pelo
  status, com um seletor "Colorir por"). Tarefas com só due_date aparecem como um marco.
- Arrastar a barra move as datas; arrastar as bordas muda início/fim. Salvar com
  atualização otimista.
- Coluna fixa à esquerda com título e avatar do responsável.
- Seção "Sem datas" no fim listando tarefas sem data, com opção de definir datas.
- Linha vertical marcando o dia de hoje; fins de semana com fundo levemente diferente.
- Respeitar os filtros ativos.
```

---

## Etapa 6 — Notas (estilo Notion)

```text
Crie a seção NOTAS, estilo Notion:

Banco (RLS com is_member()):
- notes: id uuid, parent_id uuid -> notes (nullable, on delete cascade), title text,
  icon text (emoji), content jsonb (documento do editor), position double precision,
  is_favorite boolean, created_by, updated_by, created_at, updated_at.

Interface:
- Na barra lateral, em "Notas", uma árvore de páginas (páginas dentro de páginas),
  expansível, com "+ Nova página", "+ Subpágina", renomear, mover arrastando, excluir
  (com confirmação) e uma seção "Favoritos".
- Editor de blocos usando BlockNote: títulos, parágrafos, listas, checklists, citações,
  blocos de código, tabelas, divisores e imagens (upload para o storage do backend,
  em um bucket privado acessível só a membros). Menu "/" para inserir blocos.
- Salvamento automático com debounce (~1s) e indicador "Salvando… / Salvo".
- Título grande editável e emoji clicável no topo da página.
- Mostrar "Editado por <nome> em <data>" no topo.
- Busca de notas por título e conteúdo.
```

---

## Etapa 7 — Tempo real, ligação tarefas ↔ notas e acabamento

```text
Melhorias finais:

1. Realtime: assine mudanças nas tabelas tasks, projects e notes para que todas as
   visualizações atualizem sozinhas quando outra pessoa editar. Em notas, se outra
   pessoa estiver editando a mesma página, mostre um aviso "Fulano está editando".
2. Comentários em tarefas: tabela task_comments (id, task_id, author_id, body,
   created_at) com RLS is_member(); exibir no painel lateral da tarefa.
3. Permitir mencionar/linkar uma nota dentro da descrição de uma tarefa e vice-versa.
4. Página inicial "Minha semana": minhas tarefas atrasadas, de hoje e dos próximos 7 dias.
5. Busca global com Ctrl+K (tarefas, projetos e notas).
6. Modo escuro, layout responsivo (funcionar bem em notebook e no celular),
   estados vazios amigáveis e skeletons de carregamento.
7. Revise todas as políticas de RLS e me mostre um resumo de quem pode ler/escrever
   cada tabela.
```

---

## Etapa 8 — Publicar e conectar ao GitHub

Não é prompt — é configuração no Lovable:

1. **Publish** (canto superior direito) → gera o link `https://<nome>.lovable.app`.
   Envie para a outra pessoa; ela cria conta com o email que está em `allowed_emails`.
2. **GitHub** → conecte sua conta. O Lovable cria um repositório e sincroniza nos dois
   sentidos (o que for editado lá aparece no Lovable, e vice-versa).
3. (Opcional) **Domínio próprio** em Settings → Domains.
4. No painel do backend, confira em Authentication se a confirmação de email está do
   jeito que você quer e se só os emails permitidos conseguem entrar.

---

## Checklist de testes (repetir a cada etapa)

- [ ] Login e logout funcionam; email fora da lista **não** consegue criar conta
- [ ] Abrindo em aba anônima, sem login, nenhum dado aparece
- [ ] Pessoa A cria/edita algo → pessoa B vê a mudança (após etapa 7, sem recarregar)
- [ ] Filtros aplicados continuam ao trocar de visualização
- [ ] Recarregar a página não perde dados nem filtros
