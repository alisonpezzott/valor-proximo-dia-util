# Valor no próximo dia útil

Dúvida recebida:  

> "Cara estou com uma situação em um fluxo onde preciso realizar a regra de D+1 e não estou conseguindo mano, preciso pegar o valor de previsões onde tudo que vencer aos sábados, domingos e feriados apareçam no próximo dia útil tem noção ou já fez isso alguma vez."  

_por @ramonnunes3493 em 15/01/2025_


## Criação da tabela ponte_dias_uteis  


Tendo a tabelas `dim_calendario` e `fact_vencimentos` (dei este nome aleatório) já importadas no Power BI, basta criar uma nova consulta chamada de `ponte_dias_uteis` no Power Query com o seguinte código:  

```pq
let
    // Referência da dim_calendario selecionando apenas as colunas necessárias
    Fonte = dim_calendario[[Data], [DiaUtilNumero]],
    
    // Gera uma Tabela com datas úteis
    DatasUteis = Table.RenameColumns(    // Renomeia
        Table.SelectRows(                    // Seleciona as linhas  
            Table.Buffer(Fonte),             // Na tabela bufferizada da etapa Fonte
            each [DiaUtilNumero] = 1         // Onde cada linha for dia útil
            )[[Data]],                       // Traz somente a coluna Data
        {{"Data", "DataUtil"}}           // De data para DataUtil
    ),
    
    // Faz o Join de todas as datas com as datas úteis
    Join = Table.Join(
        Fonte[[Data]], {"Data"},    // Tabela da esquerda com todas as datas
        DatasUteis, {"DataUtil"},   // Tabela da direita apenas com datas úteis
        JoinKind.LeftOuter,         // Tipo da junção. Todas as linhas da esquerda e nulo para não encontradas
        JoinAlgorithm.RightHash     // Algoritmo armazenando as datas úteis para melhor performance
    ),
    
    // Traz a próxima data útil para as linhas nulas
    PreenchidoAcima = Table.FillUp(Join,{"DataUtil"})
in
    PreenchidoAcima
```  

Ou se preferir pode utilizar o TMDL view utilize o código abaixo  

```tmdl
createOrReplace
    table ponte_dias_uteis

        column Data
            dataType: dateTime
            formatString: Short Date
            summarizeBy: none
            sourceColumn: Data

            annotation SummarizationSetBy = Automatic

            annotation UnderlyingDateTimeDataType = Date

        column DataUtil
            dataType: dateTime
            formatString: Short Date
            summarizeBy: none
            sourceColumn: DataUtil

            annotation SummarizationSetBy = Automatic

            annotation UnderlyingDateTimeDataType = Date

        partition ponte_dias_uteis = m
            mode: import
            source = ```
                    let
                        // Referência da dim_calendario selecionando apenas as colunas necessárias
                        Fonte = dim_calendario[[Data], [DiaUtilNumero]],
                        
                        // Gera uma Tabela com datas úteis
                        DatasUteis = Table.RenameColumns(    // Renomeia
                            Table.SelectRows(                    // Seleciona as linhas  
                                Table.Buffer(Fonte),             // Na tabela bufferizada da etapa Fonte
                                each [DiaUtilNumero] = 1         // Onde cada linha for dia útil
                                )[[Data]],                       // Traz somente a coluna Data
                            {{"Data", "DataUtil"}}           // De data para DataUtil
                        ),
                        
                        // Faz o Join de todas as datas com as datas úteis
                        Join = Table.Join(
                            Fonte[[Data]], {"Data"},    // Tabela da esquerda com todas as datas
                            DatasUteis, {"DataUtil"},   // Tabela da direita apenas com datas úteis
                            JoinKind.LeftOuter,         // Tipo da junção. Todas as linhas da esquerda e nulo para não encontradas
                            JoinAlgorithm.RightHash     // Algoritmo armazenando as datas úteis para melhor performance
                        ),
                        
                        // Traz a próxima data útil para as linhas nulas
                        PreenchidoAcima = Table.FillUp(Join,{"DataUtil"})
                    in
                        PreenchidoAcima
                    ```

	annotation PBI_NavigationStepName = Navegação

	annotation PBI_ResultType = Table


```  
  

## Relacionamentos  

Efetue os seguintes relacionamentos:  

`dim_calendario[Data]` **1 ----> N** `ponte_dias_uteis[DataUtil]`  
`ponte_dias_uteis[Data]` **1 ---> N** `fact_vencimentos[DataVencimento]` e deixe este inativo  

Você pode também usar o script TMDL para criar os relacionamentos  

```tmdl
createOrReplace

	relationship dim_calendario-ponte_dias_uteis
		fromColumn: ponte_dias_uteis.DataUtil
		toColumn: dim_calendario.Data

	relationship ponte_dias_uteis-fact_vencimentos
		isActive: false
		fromColumn: fact_vencimentos.DataVencimento
		toColumn: ponte_dias_uteis.Data

```  

O resultado do modelo deve ficar assim:

![Relacionamento](assets\relacionamento.png)


## Medidas  

Use a medida `Vencimentos na data` para analisar o total do valor nas datas de vencimento originais da tabela fato.

```dax
Vencimentos na data = SUM('fact_vencimentos'[Valor])
```  

Use a medida `Vencimentos na data útil` para analisar o total do valor pela própria ou próxima data útil.  

```dax
Vencimentos na data útil = 
    CALCULATE(
        SUM('fact_vencimentos'[Valor]),
        USERELATIONSHIP('ponte_dias_uteis'[Data], 'fact_vencimentos'[DataVencimento])
    )
```  

Se preferir pode usar o seguinte script TMDL para criar as novas medidas.  

```tmdl
createOrReplace

	ref table fact_vencimentos

		measure 'Vencimentos na data' = SUM('fact_vencimentos'[Valor])
			formatString: "R$"\ #,0.00;-"R$"\ #,0.00;"R$"\ #,0.00

			annotation PBI_FormatHint = {"currencyCulture":"pt-BR"}

		measure 'Vencimentos na data útil' = ```
				CALCULATE(
					SUM('fact_vencimentos'[Valor]),
					USERELATIONSHIP('ponte_dias_uteis'[Data], 'fact_vencimentos'[DataVencimento])
				)
			
		```
			formatString: "R$"\ #,0.00;-"R$"\ #,0.00;"R$"\ #,0.00

			annotation PBI_FormatHint = {"currencyCulture":"pt-BR"}
```


## Projeto PBIP  

Se preferir pode efetuar efetuar o download da pasta [PBIP]() e abrir em seu Power BI Desktop também.  

## Resultado final  

Monte um visual de tabela adicione as dimensões que desejar e coloque as duas medidas criadas para analisar o deslocamento para os dias úteis.  

![Tabela](assets\tabela.png) 






