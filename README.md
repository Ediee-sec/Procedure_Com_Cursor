### Procedure com Cursor
---

> *Recentemente me deparei com uma situação no ambiente de trabalho, onde precisei atualizar um campo do banco de dados que não estava sendo preenchido devido a um erro de lógica do sistema, o programador me solicitou que incluisse a informação neste campo em todos os casos em que este campo seja em Null ou em branco e que este campo deve ser preenchido apenas em caso de o produto estar locado no cliente, porém o programador não tinha a minima noção de qual informação deviamos colocar em cada campo para cada produto.*
---
### **Minha Analise**

> ***O Primeiro Passo*** *Foi identificar o campo que eu deveria atualizar, neste caso o **TEW_CODEQU***

> ***O Segundo Passo*** *Identificar o campo que irei pegar a informação para alimentar o **TEW_CODEQU** e neste caso é o  **TFI_COD***

> ***O Terceiro Passo*** *Identificar as condições, Neste caso é:*
> 1. O campo **TEW_CODEQU** deve ser branco ou nulo
> 2. O Produto deve estar locado no cliente

> ***O Quarto Passo*** *Realizar a Query, que é:*
~~~~sql
-- Realizando a Query para retornar os dados que devem ser atualizados 
SELECT TOP 1 TEW_PRODUT,TEW_CODEQU, TFI_COD

FROM TEW010 TW -- Tabela de Movimentações

INNER JOIN TFI010 TF (NOLOCK) ON -- Tabela de Itens do orçamento simplificado
TFI_PRODUT = TEW_PRODUT
AND TEW_FILIAL = TFI_FILIAL
AND TF.D_E_L_E_T_ = ''

WHERE			-- Sub-select para retornar apenas produtos que estão de fato no cliente e uso como condição no where, apenas produtos em cliente me interessa neste caso 
TEW_PRODUT = (SELECT  TOP 1 TEW_PRODUT FROM TEW010 WHERE TEW_CODEQU = ''
                          AND D_E_L_E_T_ = ''
                          AND TEW_PRODUT <> ''
                          AND TEW_MOTIVO NOT IN ('1','2')
                          AND TEW_QTDRET = '0' AND TEW_DTRINI != ''
                          AND TEW_DTRFIM = '')
AND TEW_CODEQU = ''
AND 1=1
AND TW.D_E_L_E_T_ = ''
ORDER BY TF.TFI_COD DESC
~~~~

> ***O Quinto Passo***, *Pensar em como podemos transformar está Query em um UPDATE, com uma analise breve podemos identificar que um simples UPDATE não resolverá este problema, um dos motivos é que o UPDATE não permite utilizar o ORDER BY e está clausula é fundamental para retornarmos apenas o registo mais recente ou o **TFI_COD** mais recente que é o correto, neste caso podemos imaginar em utilizar um UPDATE dentro de um cursor*

> ***O Sexto Passo*** *Pensar em jogar o nosso cursor dentro de uma procedure, está procedure será colocada dentro de um códio fonte do sistema, onde o Programador/Usuário poderá atualizar o campo sempre que necessário*

----

### Resultado Final

~~~sql
-- Criando a Procedure, *sem parâmetros*
CREATE PROCEDURE [dbo].[sp_Update_TEW_CODEQU]
AS

-- Declarando as Váriaveis que irei utilizar no Cursor
DECLARE             @tew_produt         VARCHAR(20)
             ,      @tew_codequ      VARCHAR(20)
             ,      @tfi_cod            VARCHAR(20)

 
-- Criando o Cursor
DECLARE cur_update_tew_codequ CURSOR FOR

-- Realizando a Query para retornar os dados que devem ser atualizados 
SELECT TOP 1 TEW_PRODUT,TEW_CODEQU, TFI_COD

FROM TEW010 TW -- Tabela de Movimentações

INNER JOIN TFI010 TF (NOLOCK) ON -- Tabela de Itens do orçamento simplificado
TFI_PRODUT = TEW_PRODUT
AND TEW_FILIAL = TFI_FILIAL
AND TF.D_E_L_E_T_ = ''

WHERE			-- Sub-select para retornar apenas produtos que estão de fato no cliente e uso como condição no where, apenas produtos em cliente me interessa neste caso 
TEW_PRODUT = (SELECT  TOP 1 TEW_PRODUT FROM TEW010 WHERE TEW_CODEQU = ''
                          AND D_E_L_E_T_ = ''
                          AND TEW_PRODUT <> ''
                          AND TEW_MOTIVO NOT IN ('1','2')
                          AND TEW_QTDRET = '0' AND TEW_DTRINI != ''
                          AND TEW_DTRFIM = '')
AND TEW_CODEQU = ''
AND 1=1
AND TW.D_E_L_E_T_ = ''
ORDER BY TF.TFI_COD DESC

-- Abrindo o cursor
OPEN cur_update_tew_codequ

-- Realizando a pesquisa dentro da Query e substituido os valores dos campos pelas váriveis declaradas
FETCH NEXT FROM cur_update_tew_codequ INTO @tew_produt, @tew_codequ, @tfi_cod

-- Loop, enquanto não for o final da tabela executamos o cursor
WHILE @@FETCH_STATUS = 0

		-- Atualiza o campo TEW_CODEQU caso ele seja em branco com base no campo TFI_COD, onde passo os valores para as váriaveis
       BEGIN
             BEGIN TRANSACTION
                    UPDATE TEW010
                    SET TEW_CODEQU = @tfi_cod
                    WHERE TEW_PRODUT = @tew_produt AND D_E_L_E_T_ = ''
         
			-- Se afetar 1 linha realizamos o COMMIT se não realizamos o ROLLBACK
             IF @@ROWCOUNT = 1
				COMMIT
             ELSE
				ROLLBACK

			-- Indo para o próximo registo do cursor
             FETCH NEXT FROM cur_update_tew_codequ INTO @tew_produt, @tew_codequ, @tfi_cod

       END

-- Fechando o cursor e liberando espaço em memória
CLOSE cur_update_tew_codequ
DEALLOCATE cur_update_tew_codequ
~~~
