# Remesh Ultimate 1.7.6 DEV — Relatório de correções e validação

**Base:** Remesh Ultimate 1.7.5 DEV (comparada com a 1.6.8)
**Entrega:** `Remesh_Ultimate_1.7.6_DEV.zip`
**Ambiente de teste:** Blender 5.2.2 (módulo Python), modelos de teste + CHASSI real do kart
**Data:** 13/11/2026

---

## 1. Arquivos modificados

| Arquivo | Situação |
|---|---|
| `remesh_ultimate/__init__.py` | Modificado (único módulo do complemento) |
| `LEIA-ME_1.7.6_DEV.txt` | Novo |
| `LEIA-ME_1.7.4_DEV.txt`, `LEIA-ME_1.7.5_DEV.txt` | Mantidos |

A estrutura do ZIP é a mesma da 1.7.5. Nenhum recurso foi removido.

---

## 2. Causas dos problemas

Comparei o código da 1.7.5 com a 1.6.8. Da 1.7.2 à 1.7.4 só tive acesso às notas do `LEIA-ME`, não ao código.

| Problema | Causa real |
|---|---|
| Cores acinzentadas e desbotadas | A paleta gerava valores HSV pensados como cor de tela (sRGB), mas o Blender guarda cor de objeto e de material como cor **linear**. Na tela tudo saía clareado, num tom pastel. Subir saturação e valor no HSV (o que a 1.7.4/1.7.5 fizeram) não corrige isso. |
| Peças espelhadas com cores diferentes entre objetos | Cada objeto tinha a sua própria paleta, começando da cor 0. Não existia nenhuma comparação entre objetos. Por isso os pares (rodas, caixas laterais) saíam com cores diferentes, e peças diferentes (roda dianteira × traseira) saíam com a mesma cor. |
| Solid "envernizado" | O modo Solid usa o *Roughness* de **Viewport Display** do material (0,4 por padrão no Blender), e não o Roughness do shader (0,88 na 1.7.4). |
| Configurações do viewport sem efeito | Só eram aplicadas se a vista já estivesse em Solid. Em Wireframe, ou rodando via script/MCP sem tela ativa, nada era aplicado. |
| Separate Parts desmarcado | O valor fica salvo dentro do `.blend`, então arquivos antigos mantinham OFF mesmo com o novo padrão. |

---

## 3. Alterações realizadas

- **Paleta nova:** 36 cores fixas e determinísticas, definidas em sRGB e convertidas para linear antes de serem gravadas.
- **Simetria em espaço de mundo:** o plano de simetria é procurado para a seleção inteira, e objetos independentes são pareados entre si (seção 4).
- **Grupo de cor compartilhado:** o índice de cor é decidido **depois** do pareamento e guardado no objeto de áreas (`ru_gcolor`). Objetos e materiais usam esse índice. Os pares usam o mesmo material (`RU_Color_NN`) e exatamente o mesmo valor de cor.
- **Wireframe:**
  - Cada parte desenha os fios com a cor do objeto.
  - A cor do fio é um tom claro da mesma cor da paleta: fios claros como na 1.6.8, mas com cores mais distintas entre si.
  - Pares espelhados têm o mesmo valor.
- **Solid:** cor do material, destaque especular desligado e materiais foscos (Roughness de viewport 1,0, Metallic 0).
- **Viewport:**
  - As configurações são aplicadas em todas as vistas 3D, inclusive sem tela ativa.
  - O estado anterior é guardado uma vez, e o botão **Restore View** (seção *Result*) devolve tudo como estava.
  - As preferências globais e o gerenciamento de cor da cena **não** são alterados.
- **Padrões:**
  - *Color Parts* e *Separate Parts* vêm ligados.
  - Arquivos antigos recebem os dois ligados uma única vez (marca `ru_defaults_176` na cena).
  - Depois disso, a escolha do usuário é respeitada.
- **Nova opção:** *Advanced → Mirror Tolerance* (padrão 3% do tamanho da peça).
- **Objeto girado:** se os eixos do mundo falham, a simetria dentro do objeto usa os eixos do próprio objeto. Quando existe plano da seleção, ele tem prioridade.
- **QuadriFlow:** densidade, projeção, cobertura e geração de quads **sem alteração**.

---

## 4. Algoritmo de identificação de simetria

1. **Peças em mundo.** Cada objeto selecionado vira uma "peça" em coordenadas de mundo, com posição, rotação e escala aplicadas. Cada peça guarda pontos de amostra, uma BVH das faces, a área e a caixa envolvente.
2. **Plano da seleção.**
   - Normais candidatas: os eixos X/Y/Z do mundo e os eixos do objeto ativo (para o caso de modelo girado na cena).
   - Posições candidatas: meio do modelo, origem do mundo e origem do objeto ativo.
   - Para cada candidato, os pontos espelhados de cada peça são medidos até a **superfície** de todas as peças, com tolerância de 1% do tamanho do conjunto.
   - A nota é ponderada pela área de cada peça, para que uma peça diferente não esconda a simetria do conjunto.
   - O plano é aceito com nota de 80% ou mais. **Sem plano confiável, nenhum par é criado.**
3. **Peças centrais.** As peças que cruzam o plano e espelham sobre si mesmas (capô, asa) são tratadas como centrais.
4. **Pares candidatos.**
   - Filtros: tamanho entre 0,75× e 1,33×, área entre 0,6× e 1,67×, e centro espelhado próximo.
   - Medida decisiva: o percentil 90 da distância entre a superfície espelhada de A e a de B (e de B para A), em % do tamanho da peça. Ela precisa ficar abaixo da *Mirror Tolerance*.
   - **Tamanho parecido sozinho nunca forma par.**
5. **Pareamento 1-para-1.** Os pares são fechados em ordem crescente de erro, e cada peça entra em no máximo um par.

---

## 5. Como os grupos compartilhados recebem cor

1. Dentro de cada objeto, os gêmeos (`.L`/`.R`) já formam um único grupo.
2. Os objetos são processados em ordem de nome. Cada objeto **líder** recebe uma faixa consecutiva de índices de cor, começando pela maior área.
3. O objeto espelhado (**seguidor**) recebe, área por área, a cor da área do líder onde cai a sua face espelhada. É um voto ponderado por área, e vence a cor com maioria de pelo menos 50%.
4. Áreas sem correspondência recebem uma cor nova. Nunca é criado um par inventado.
5. A cor de cada área é `PALETA[índice]`: o mesmo índice dá **exatamente** o mesmo valor.

---

## 6. Valores da nova paleta (sRGB, em ordem de uso)

```
FF0D0D 0DFF0D 267DFF 20D6C4 FFD426 FF0DB6 C25762 90D660 B340FF 26BEFF C2660A AC57C2
FF0D6E C2AC57 BEEB0C FF73B9 23EB9B 0AC21C FF9D73 FF5959 FF0DFF 6084D6 FF73FF FF9E0D
B3C230 FF6726 9D73FF FFBD59 7BEB0C 57C282 C29D0A 3BEB6F 78C20A C28344 D620C4 EB5280
```

- **Como foram escolhidas:** cada cor é a mais distante possível, em CIELAB, de todas as escolhidas antes dela.
- **Limites:** luminosidade L\* entre 51 e 88 e croma de pelo menos 45. Não há cores esbranquiçadas nem escuras demais.
- **Menor diferença entre duas cores (dE):**

| Cores | Paleta antiga | Paleta nova |
|---|---|---|
| 12 primeiras | 29 | **41** |
| 24 primeiras | 11 | **27** |
| 36 | — | **20** |

- **Acima de 36:** matizes pelo ângulo áureo.
- **Fios (Wireframe):** a mesma cor misturada 50% em direção ao branco, com L\* entre 69 e 93.

---

## 7. Configurações de Wireframe e Solid alteradas

Todas são guardadas antes da alteração e podem ser desfeitas com **Restore View**.

| Configuração | Valor |
|---|---|
| `shading.color_type` | `MATERIAL` |
| `shading.wireframe_color_type` | `OBJECT` |
| `shading.show_specular_highlight` | `False` |
| `shading.light` | `STUDIO` |
| `overlay.wireframe_opacity` | `1.0` |

**Materiais RU:**
- `diffuse_color` recebe a cor da paleta.
- Roughness de viewport 1,0 e Metallic 0.
- No Principled BSDF: Roughness 0,88 e Specular 0,12.

---

## 8. Resultado de cada teste

| Teste | Descrição | Resultado |
|---|---|---|
| **A** | Peças espelhadas no mesmo objeto (kart, Suzanne, kart não-manifold) | 11/11, 13/13 e 27/27 pares com a mesma cor — **APROVADO** |
| **B** | Objetos independentes (kart separado em 10 objetos, incluindo 4 rodas) | 24/24 pares com a mesma cor, 14 deles entre objetos diferentes. Rodas esquerda × direita iguais; dianteira × traseira diferentes — **APROVADO** |
| B (1.7.5) | Mesma medição na versão anterior | 10/24 pares, 0/14 entre objetos, rodas dianteira e traseira com a mesma cor — **falha confirmada** |
| **C** | Modelo deslocado para (40, −25, 7) | 24/24 — **APROVADO** |
| C extra | Deslocado e girado 35° | 20/20 — **APROVADO** |
| **D** | Pequenas diferenças (caixa lateral com ruído de cerca de 1%) | 24/24 — **APROVADO** |
| D negativo | Roda dianteira esquerda trocada por uma caixa do mesmo tamanho | **Não** pareou com a roda direita (correto); os outros 13 pares entre objetos continuam — **APROVADO** |
| **E** | Distribuição e determinismo | 31 cores para 57 partes; na segunda execução, 57/57 partes com a mesma cor — **APROVADO** |
| **F** | Wireframe | Verificado por simulação (cores dos fios desenhadas sobre o cinza do viewport) e com um viewport simulado (aplicar e restaurar). **Não verificado no Blender da usuária**: a conexão não respondeu nesta sessão. |
| **G** | Solid | Verificado por render Workbench (o mesmo motor do viewport): cores vivas e sem brilho especular. **Não verificado no Blender da usuária**, pelo mesmo motivo. |
| **H** | Configurações padrão | Cena nova: ambos ligados. Arquivo antigo com ambos OFF: ligados uma vez. Se o usuário desliga depois, a escolha é mantida — **APROVADO** |
| **I** | Regressão do QuadriFlow | 10 casos (kart, kart não-manifold, Suzanne, Suzanne densa com ruído, CHASSI real; métodos Smart e Rebuild): faces, polos, cobertura e desvio **idênticos** aos da 1.7.5 — **APROVADO** |

**Regressão adicional:** funcionando Detect Areas, Dense to Collection, Remesh Areas, REMESH com malha única colorida, vários objetos de uma vez, arquivo antigo e instalação a partir do ZIP.

---

## 9. Erros encontrados e corrigidos durante os testes

1. **Plano da seleção:** a detecção comparava pontos com pontos e falhava numa peça grande com poucas amostras. Agora mede a distância até a superfície.
2. **Uma peça diferente derrubava todos os pares** (por exemplo, a roda trocada por uma caixa). A nota do plano passou a ser ponderada pela área de cada peça.
3. **Modelo girado:** a asa, que é uma lâmina fina, achava simetria de cima para baixo em vez de esquerda e direita. Agora o plano da seleção tem prioridade e, sem ele, valem os eixos do próprio objeto.

---

## 10. Limitações

- O pareamento entre objetos só acontece entre objetos **processados juntos**, selecionados na mesma execução.
- Peças espelhadas muito diferentes (acima da *Mirror Tolerance*) não pareiam. Isso é proposital.
- Acima de 36 grupos as cores continuam distintas, mas com contraste menor.
- O gerenciamento de cor da cena (AgX) não é alterado. Ele reduz um pouco a saturação no Material Preview e no Rendered.
- Os testes F e G ainda precisam ser confirmados no Blender da usuária.
