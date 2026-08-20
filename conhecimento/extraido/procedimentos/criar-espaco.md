# Procedimento: criar e configurar um Espaço (Workspace)

**Fonte:** `m1:46-50` `[UI-OBSERVADA]` / `[VÍDEO]` — a transcrição não tem timestamp.
**Pré-requisito:** `[VÍDEO]` ser **administrador**. Criar espaço é ação restrita a admin (`m1:47`).

---

1. **Abra o seletor de contexto do espaço**, no ícone do espaço.
   → Deve aparecer o espaço atual (rotulado **"WS <nome>"**) e a lista dos demais espaços aos
   quais você tem acesso. No rodapé, a organização.

2. **Clique para criar novo espaço.**
   → Abre um modal de cadastro.

3. **Informe o nome** e confirme.
   → O espaço passa a aparecer na lista.

4. **Abra a edição de propriedades** pelo ícone de lápis (canetinha).
   → Abre a tela de propriedades do espaço.

5. **Defina os sistemas que existem nesse espaço.**
   → Aqui entram o sistema próprio **e os concorrentes**, cada um com o banco que usa (ex.:
   ConstruShow com seu banco; um concorrente como SQL Server).
   *Por que importa:* é o que permite depois identificar a origem numa migração.

6. **(Opcional) Defina cor e ícone.**
   → A cor é aplicada na interface do espaço imediatamente. O ícone pode ser um predefinido ou
   upload próprio.
   *Convenção observada:* usa-se a cor da vertical (ex.: verde para Agro).

7. **(Opcional) Renomeie**, se houver erro de escrita. O campo organização é somente leitura.

8. **(Opcional) Marque o espaço como favorito** clicando na estrela, na lista de espaços.
   → No próximo login, entra direto nesse espaço. **Sem favorito, entra no primeiro da lista.**

---

## Observações

`[VÍDEO]` O critério real de criação de espaço na Viasoft é **vertical de ERP**, não time:
`ConstruShow`, `Agrotitan`, `Siagri`, `PetroShow`, `Talent` (`m1:44-45`). Isso divergindo
levemente de `conceitos.md § Espaço` `[CONFIRMADO-DOC]`, que define espaço como "um produto ou
time".

`[VÍDEO]` **Não há transferência de fluxo entre espaços** (`m1:58`). O contorno é exportar e
importar. Ver `procedimentos/criar-fluxo.md`.

`[LACUNA]` Não foi demonstrado como **conceder acesso** de outro usuário ao espaço recém-criado.
A aula `m2p2` cobre isso (Configurações → perfil Administrador) e será extraída no lote 2.
