Santander Product Recommendation-3rd Place Soulution
================

KAGGLE競賽
----------

這個競賽資料為西班牙Santander銀行的客戶樣態模擬資料，包含1.5年(2015-01-28 ~ 2016-05-28)的客戶基本資訊以及產品持有狀態，各欄位說明請見[資料描述](https://www.kaggle.com/c/santander-product-recommendation/data)。<br> 競賽目標分析目標是根據這份資料，預測每個客戶在2016-06-28的時候，除了現已持有的商品外，去推薦客戶七項商品後，根據是否有購買作為驗證。<br>

這裡透過比賽完參賽者分享的程式碼進行說明，這個參賽者分享的程式碼是利用 2016-05-28 進行預測的，使用的演算法為XGBoost。 data產生出的結果Private Score：0.0312031。Public Score：0.0308417。<br>

先進行簡單說明匯總：<br> 1.將要預測的24項商品分成三類：<br> (1)將只有少數人持有的商品(ind\_ahor\_fin\_ult1, ind\_aval\_fin\_ult1, ind\_deco\_fin\_ult1 and ind\_deme\_fin\_ult1)給定固定機率值**1^-10**。<br> (2)使用固定月份進行預測：ind\_cco\_fin\_ult1僅用2015-12-28的資料進行預測、ind\_reca\_fin\_ult1僅用2015-06-28的資料進行預測。<br> (3)其他18項商品分別預測2016-05-28, 2016-04-28, 2016-03-28, 2016-02-28, 2016-01-28和 2015-12-28這六個月，但後面跑XGBoost其實只有用到**2016-05-28**，並且在預測的部分，僅使用該月新持有該產品的客戶。<br> ![](overview.png)

2.總共使用了142個特徵變數，分成幾個部分：<br> (1) 18個原始特徵變數：排除了fecha\_dato(資料年月)，ncodpers(客代)，fecha\_alta，ult\_fec\_cli\_1t，tipodom(地址類型)和cod\_prov(洲代碼)。<br> (2) 2個串聯(concatenation)欄位，當月與前一個月串聯：ind\_actividad\_cliente(是否活躍)、tiprel\_1mes。<br> (3) 20個特徵：20個商品**上個月**的持有狀況。<br> (4) 1個串聯(concatenation)欄位：將上個月的20個商品串成一個字元。<br> (5) 1個特徵：上個月20個商品購買的數量。<br> (6) 80個特徵：20個商品從最初到預測前一個月index狀態變更的計數(count)，共4種狀態變更\*20個商品。<br> (7) 20個特徵：20個商品直到預測前一個月時連續未持有的月份數。<br>

名詞解釋：<br> 1. 串聯(concatenation)：將某一個特徵變數這個月與前一個月用串聯的方式合併再一起。例如：如果某個月的“tiprel\_1mes”=“A”，而上個月的“tiprel\_1mes”=“I”，則它們的串聯後為新特徵為"AI"。<br> 2. index狀態變更：0to0(前次無持有到這次無持有)，0to1(前次無持有到這次持有)，1to0(前次持有到這次無持有)和1to1(前次持有到這次持有)。<br>

程式碼說明
----------

將程式碼分成**三個部分**，分別做說明：

### 呼叫套件

``` r
library(data.table)
library(dplyr)
library(xgboost)
```

1\_make\_data\_v3
-----------------

``` r
#先排除沒有使用的變數fecha_alta,ult_fec_cli_1t,tipodom,cod_prov。
#還有四個使用固定預測機率為1^-10的商品(ind_ahor_fin_ult1,ind_aval_fin_ult1,ind_deco_fin_ult1,ind_deme_fin_ult1)也進行排除。
#並強制indrel_1mes、conyuemp為字元。
data <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/train_ver2.csv", drop=c(7,11,19,20,25,26,34,35), colClasses=c(indrel_1mes="character", conyuemp="character"))
```

    ## 
    Read 0.0% of 13647309 rows
    Read 3.3% of 13647309 rows
    Read 6.5% of 13647309 rows
    Read 9.8% of 13647309 rows
    Read 13.0% of 13647309 rows
    Read 16.3% of 13647309 rows
    Read 19.6% of 13647309 rows
    Read 22.9% of 13647309 rows
    Read 26.2% of 13647309 rows
    Read 29.5% of 13647309 rows
    Read 32.7% of 13647309 rows
    Read 36.0% of 13647309 rows
    Read 39.3% of 13647309 rows
    Read 42.6% of 13647309 rows
    Read 45.9% of 13647309 rows
    Read 49.3% of 13647309 rows
    Read 52.2% of 13647309 rows
    Read 55.3% of 13647309 rows
    Read 58.5% of 13647309 rows
    Read 61.7% of 13647309 rows
    Read 64.8% of 13647309 rows
    Read 68.3% of 13647309 rows
    Read 71.7% of 13647309 rows
    Read 75.1% of 13647309 rows
    Read 78.4% of 13647309 rows
    Read 81.8% of 13647309 rows
    Read 85.2% of 13647309 rows
    Read 88.6% of 13647309 rows
    Read 91.8% of 13647309 rows
    Read 95.2% of 13647309 rows
    Read 98.6% of 13647309 rows
    Read 13647309 rows and 40 (of 48) columns from 2.135 GB file in 00:00:43

``` r
#取資料的年月出來，並去重複+加上要預測的"2016-06-28"進來。
date.list <- c(unique(data$fecha_dato), "2016-06-28")
print(date.list)
```

    ##  [1] "2015-01-28" "2015-02-28" "2015-03-28" "2015-04-28" "2015-05-28"
    ##  [6] "2015-06-28" "2015-07-28" "2015-08-28" "2015-09-28" "2015-10-28"
    ## [11] "2015-11-28" "2015-12-28" "2016-01-28" "2016-02-28" "2016-03-28"
    ## [16] "2016-04-28" "2016-05-28" "2016-06-28"

``` r
#將商品列表出來 colnames商品名稱，ncol欄位數，因為商品的部分是從後面數過來順序排列，所以直接取最後的20欄。
product.list <- colnames(data)[(ncol(data)-19):ncol(data)]
print(product.list)
```

    ##  [1] "ind_cco_fin_ult1"  "ind_cder_fin_ult1" "ind_cno_fin_ult1" 
    ##  [4] "ind_ctju_fin_ult1" "ind_ctma_fin_ult1" "ind_ctop_fin_ult1"
    ##  [7] "ind_ctpp_fin_ult1" "ind_dela_fin_ult1" "ind_ecue_fin_ult1"
    ## [10] "ind_fond_fin_ult1" "ind_hip_fin_ult1"  "ind_plan_fin_ult1"
    ## [13] "ind_pres_fin_ult1" "ind_reca_fin_ult1" "ind_tjcr_fin_ult1"
    ## [16] "ind_valo_fin_ult1" "ind_viv_fin_ult1"  "ind_nomina_ult1"  
    ## [19] "ind_nom_pens_ult1" "ind_recibo_ult1"

``` r
##### data 1: inner join with last month #####
# i取6,12,13,14,15,16,17,18，表示在這迴圈中處理這幾個月，"2015-06-28" "2015-12-28" "2016-01-28" "2016-02-28" "2016-03-28" "2016-04-28" "2016-05-28" "2016-06-28"這幾個月的資料。

#for(i in c(6,12:length(date.list))) {

for(i in c(17:length(date.list))) {
  
  #print(date.list[i])
#除了最後一個要預測的月份，根據客代去merge前一個月的值，取出前一個月的ind_actividad_cliente(是否活躍)、tiprel_1mes(月初的客戶關係類型)、20項商品這些欄位並且加上_last的后綴。
  
  if(date.list[i] != "2016-06-28") {
    out <- data[fecha_dato==date.list[i]]
    out <- merge(out, data[fecha_dato==date.list[i-1], c("ncodpers","tiprel_1mes","ind_actividad_cliente",product.list), with=FALSE], by="ncodpers", suffixes=c("","_last"))
    #write.csv(out, paste0("train_", date.list[i], ".csv"), row.names=FALSE)
  } else {
    
#最後要預測的月份，利用test資料集，排除沒有使用的變數fecha_alta, ult_fec_cli_1t, tipodom, cod_prov，後照上面一樣的處理方式，差別在不同的資料來源。
    
    out <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/test_ver2.csv", drop=c(7,11,19,20), colClasses=c(indrel_1mes="character", conyuemp="character"))
    out <- merge(out, data[fecha_dato==date.list[i-1], c("ncodpers","tiprel_1mes","ind_actividad_cliente",product.list), with=FALSE], by="ncodpers", suffixes=c("","_last"))
    colnames(out)[(ncol(out)-19):ncol(out)] <- paste0(colnames(out)[(ncol(out)-19):ncol(out)], "_last")
    #write.csv(out, paste0("test_", date.list[i], ".csv"), row.names=FALSE)
  }
}
#前一個月份的相關變數處理。
data1 <-fread("C:/Users/aa006/Desktop/3rd Place Soulution/train_2016-05-28.csv", header = T)
```

    ## 
    Read 45.3% of 926663 rows
    Read 83.1% of 926663 rows
    Read 926663 rows and 62 (of 62) columns from 0.174 GB file in 00:00:04

``` r
head(data1)
```

    ##    ncodpers fecha_dato ind_empleado pais_residencia sexo age ind_nuevo
    ## 1:    15889 2016-05-28            F              ES    V  56         0
    ## 2:    15890 2016-05-28            A              ES    V  63         0
    ## 3:    15892 2016-05-28            F              ES    H  62         0
    ## 4:    15893 2016-05-28            N              ES    V  63         0
    ## 5:    15894 2016-05-28            A              ES    V  60         0
    ## 6:    15895 2016-05-28            A              ES    H  50         0
    ##    antiguedad indrel indrel_1mes tiprel_1mes indresi indext conyuemp
    ## 1:        255      1           1           A       S      N        N
    ## 2:        256      1           1           A       S      N        N
    ## 3:        256      1           1           A       S      N        N
    ## 4:        256      1           1           A       S      N        N
    ## 5:        256      1           1           A       S      N        N
    ## 6:        256      1           1           A       S      N        N
    ##    canal_entrada indfall nomprov ind_actividad_cliente    renta
    ## 1:           KAT       N  MADRID                     1 326124.9
    ## 2:           KAT       N  MADRID                     1  71461.2
    ## 3:           KAT       N  MADRID                     1 430477.4
    ## 4:           KAT       N  MADRID                     1 430477.4
    ## 5:           KAT       N  MADRID                     1 281757.7
    ## 6:           KAT       N  MADRID                     1 205427.6
    ##             segmento ind_cco_fin_ult1 ind_cder_fin_ult1 ind_cno_fin_ult1
    ## 1:          01 - TOP                1                 0                0
    ## 2:          01 - TOP                0                 0                1
    ## 3:          01 - TOP                1                 0                0
    ## 4: 02 - PARTICULARES                0                 0                0
    ## 5:          01 - TOP                1                 0                0
    ## 6: 02 - PARTICULARES                1                 0                0
    ##    ind_ctju_fin_ult1 ind_ctma_fin_ult1 ind_ctop_fin_ult1 ind_ctpp_fin_ult1
    ## 1:                 0                 0                 0                 1
    ## 2:                 0                 0                 0                 1
    ## 3:                 0                 0                 0                 0
    ## 4:                 0                 0                 0                 0
    ## 5:                 0                 0                 0                 0
    ## 6:                 0                 0                 0                 0
    ##    ind_dela_fin_ult1 ind_ecue_fin_ult1 ind_fond_fin_ult1 ind_hip_fin_ult1
    ## 1:                 0                 0                 0                0
    ## 2:                 0                 1                 0                0
    ## 3:                 1                 1                 0                0
    ## 4:                 0                 0                 0                0
    ## 5:                 0                 1                 0                0
    ## 6:                 0                 1                 0                0
    ##    ind_plan_fin_ult1 ind_pres_fin_ult1 ind_reca_fin_ult1 ind_tjcr_fin_ult1
    ## 1:                 0                 0                 0                 1
    ## 2:                 1                 0                 0                 1
    ## 3:                 0                 0                 1                 1
    ## 4:                 0                 0                 0                 0
    ## 5:                 0                 0                 1                 1
    ## 6:                 1                 0                 1                 1
    ##    ind_valo_fin_ult1 ind_viv_fin_ult1 ind_nomina_ult1 ind_nom_pens_ult1
    ## 1:                 1                0               0                 0
    ## 2:                 0                0               1                 1
    ## 3:                 1                0               0                 0
    ## 4:                 1                0               0                 0
    ## 5:                 1                0               1                 1
    ## 6:                 1                0               0                 0
    ##    ind_recibo_ult1 tiprel_1mes_last ind_actividad_cliente_last
    ## 1:               0                A                          1
    ## 2:               1                A                          1
    ## 3:               1                A                          1
    ## 4:               0                A                          1
    ## 5:               1                A                          1
    ## 6:               1                A                          1
    ##    ind_cco_fin_ult1_last ind_cder_fin_ult1_last ind_cno_fin_ult1_last
    ## 1:                     1                      0                     0
    ## 2:                     0                      0                     1
    ## 3:                     1                      0                     0
    ## 4:                     0                      0                     0
    ## 5:                     1                      0                     0
    ## 6:                     1                      0                     0
    ##    ind_ctju_fin_ult1_last ind_ctma_fin_ult1_last ind_ctop_fin_ult1_last
    ## 1:                      0                      0                      0
    ## 2:                      0                      0                      0
    ## 3:                      0                      0                      0
    ## 4:                      0                      0                      0
    ## 5:                      0                      0                      0
    ## 6:                      0                      0                      0
    ##    ind_ctpp_fin_ult1_last ind_dela_fin_ult1_last ind_ecue_fin_ult1_last
    ## 1:                      1                      0                      0
    ## 2:                      1                      0                      1
    ## 3:                      0                      1                      1
    ## 4:                      0                      0                      0
    ## 5:                      0                      0                      1
    ## 6:                      0                      0                      1
    ##    ind_fond_fin_ult1_last ind_hip_fin_ult1_last ind_plan_fin_ult1_last
    ## 1:                      0                     0                      0
    ## 2:                      0                     0                      1
    ## 3:                      0                     0                      0
    ## 4:                      0                     0                      0
    ## 5:                      0                     0                      0
    ## 6:                      0                     0                      1
    ##    ind_pres_fin_ult1_last ind_reca_fin_ult1_last ind_tjcr_fin_ult1_last
    ## 1:                      0                      0                      0
    ## 2:                      0                      0                      1
    ## 3:                      0                      1                      1
    ## 4:                      0                      0                      0
    ## 5:                      0                      1                      1
    ## 6:                      0                      1                      1
    ##    ind_valo_fin_ult1_last ind_viv_fin_ult1_last ind_nomina_ult1_last
    ## 1:                      1                     0                    0
    ## 2:                      0                     0                    1
    ## 3:                      1                     0                    0
    ## 4:                      1                     0                    0
    ## 5:                      1                     0                    1
    ## 6:                      1                     0                    0
    ##    ind_nom_pens_ult1_last ind_recibo_ult1_last
    ## 1:                      0                    0
    ## 2:                      1                    1
    ## 3:                      0                    1
    ## 4:                      0                    0
    ## 5:                      1                    1
    ## 6:                      0                    1

``` r
##### data 2: count the change of index #####
#for(i in c(6,12:length(date.list))) {
for(i in c(17:length(date.list))) {
  
  #print(date.list[i])
#if取出當月與前一個月的客代相同的客代，除了最後一個要預測的月份，根據客代去merge前一個月的客代，要預測的月份則是利用test資料集，取客代出來。 
  
  if(date.list[i] != "2016-06-28") {
    out <- merge(data[fecha_dato==date.list[i], .(ncodpers)], data[fecha_dato==date.list[i-1], .(ncodpers)], by="ncodpers")
  } else {
    out <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/test_ver2.csv", select=2)
  }

  for(product in product.list) {
    
    #print(product)
#找出前一個月以前到最初(1~i-1)的月份、客代、產品。
    
    temp <- data[fecha_dato %in% date.list[1:(i-1)], c("fecha_dato","ncodpers",product), with=FALSE]

#根據客代 、年月的順序排序。
    
    temp <- temp[order(ncodpers, fecha_dato)]
    
#根據該期與該期減一期去看持有某一項產品的狀態，分成四個欄位(00，01，10，11)，給定TURE、FALSE。  
    
    temp$n00 <- temp$ncodpers==lag(temp$ncodpers) & lag(temp[[product]])==0 & temp[[product]]==0
    temp$n01 <- temp$ncodpers==lag(temp$ncodpers) & lag(temp[[product]])==0 & temp[[product]]==1
    temp$n10 <- temp$ncodpers==lag(temp$ncodpers) & lag(temp[[product]])==1 & temp[[product]]==0
    temp$n11 <- temp$ncodpers==lag(temp$ncodpers) & lag(temp[[product]])==1 & temp[[product]]==1
    
#因為每個客戶不是都從一開始就進來，所以要補0。
    
    temp[is.na(temp)] <- 0
    
#根據客代分別加總四個狀態的值。
    
    count <- temp[, .(sum(n00, na.rm=TRUE), sum(n01, na.rm=TRUE), sum(n10, na.rm=TRUE), sum(n11, na.rm=TRUE)), by=ncodpers]
    colnames(count)[2:5] <- paste0(product, c("_00","_01","_10","_11"))
    
#先加了一個_0len的欄位，裡面都先放0，再跑迴圈，計算出連續未持有的月份數。
    
    count[[paste0(product,"_0len")]] <- 0
    for(date in date.list[1:(i-1)]) {
      temp2 <- temp[fecha_dato==date]
      temp2 <- temp2[match(count$ncodpers, ncodpers)]
      flag <- temp2[[product]] == 0
      flag[is.na(flag)] <- 0
      count[[paste0(product,"_0len")]] <- (count[[paste0(product,"_0len")]] + 1) * flag
    }
    out <- merge(out, count, by="ncodpers")
  }
  #write.csv(out[, -1, with=FALSE], paste0("count_", date.list[i], ".csv"), quote=FALSE, row.names=FALSE)
}
```

``` r
data2 <-fread("C:/Users/aa006/Desktop/3rd Place Soulution/count_2016-05-28.csv", header = T)
```

    ## 
    Read 25.9% of 926663 rows
    Read 47.5% of 926663 rows
    Read 69.1% of 926663 rows
    Read 93.9% of 926663 rows
    Read 926663 rows and 100 (of 100) columns from 0.199 GB file in 00:00:06

``` r
select(filter(data,ncodpers==15889), fecha_dato,ncodpers,ind_cco_fin_ult1)
```

    ##    fecha_dato ncodpers ind_cco_fin_ult1
    ## 1  2015-01-28    15889                1
    ## 2  2015-02-28    15889                1
    ## 3  2015-03-28    15889                1
    ## 4  2015-04-28    15889                1
    ## 5  2015-05-28    15889                1
    ## 6  2015-06-28    15889                1
    ## 7  2015-07-28    15889                1
    ## 8  2015-08-28    15889                1
    ## 9  2015-09-28    15889                1
    ## 10 2015-10-28    15889                1
    ## 11 2015-11-28    15889                1
    ## 12 2015-12-28    15889                1
    ## 13 2016-01-28    15889                1
    ## 14 2016-02-28    15889                1
    ## 15 2016-03-28    15889                1
    ## 16 2016-04-28    15889                1
    ## 17 2016-05-28    15889                1

``` r
select(data2[1],ind_cco_fin_ult1_00,ind_cco_fin_ult1_01,ind_cco_fin_ult1_10,ind_cco_fin_ult1_11,ind_cco_fin_ult1_0len)
```

    ##    ind_cco_fin_ult1_00 ind_cco_fin_ult1_01 ind_cco_fin_ult1_10
    ## 1:                   0                   0                   0
    ##    ind_cco_fin_ult1_11 ind_cco_fin_ult1_0len
    ## 1:                  15                     0

``` r
#共產生了100個變數。
head(data2)
```

    ##    ind_cco_fin_ult1_00 ind_cco_fin_ult1_01 ind_cco_fin_ult1_10
    ## 1:                   0                   0                   0
    ## 2:                  15                   0                   0
    ## 3:                   4                   1                   0
    ## 4:                  15                   0                   0
    ## 5:                   0                   0                   0
    ## 6:                   0                   0                   0
    ##    ind_cco_fin_ult1_11 ind_cco_fin_ult1_0len ind_cder_fin_ult1_00
    ## 1:                  15                     0                   15
    ## 2:                   0                    16                   15
    ## 3:                  10                     0                   15
    ## 4:                   0                    16                   15
    ## 5:                  15                     0                   15
    ## 6:                  15                     0                   15
    ##    ind_cder_fin_ult1_01 ind_cder_fin_ult1_10 ind_cder_fin_ult1_11
    ## 1:                    0                    0                    0
    ## 2:                    0                    0                    0
    ## 3:                    0                    0                    0
    ## 4:                    0                    0                    0
    ## 5:                    0                    0                    0
    ## 6:                    0                    0                    0
    ##    ind_cder_fin_ult1_0len ind_cno_fin_ult1_00 ind_cno_fin_ult1_01
    ## 1:                     16                  15                   0
    ## 2:                     16                   0                   0
    ## 3:                     16                  10                   0
    ## 4:                     16                  15                   0
    ## 5:                     16                   7                   3
    ## 6:                     16                  10                   0
    ##    ind_cno_fin_ult1_10 ind_cno_fin_ult1_11 ind_cno_fin_ult1_0len
    ## 1:                   0                   0                    16
    ## 2:                   0                  15                     0
    ## 3:                   1                   4                    11
    ## 4:                   0                   0                    16
    ## 5:                   4                   1                     6
    ## 6:                   1                   4                    11
    ##    ind_ctju_fin_ult1_00 ind_ctju_fin_ult1_01 ind_ctju_fin_ult1_10
    ## 1:                   15                    0                    0
    ## 2:                   15                    0                    0
    ## 3:                   15                    0                    0
    ## 4:                   15                    0                    0
    ## 5:                   15                    0                    0
    ## 6:                   15                    0                    0
    ##    ind_ctju_fin_ult1_11 ind_ctju_fin_ult1_0len ind_ctma_fin_ult1_00
    ## 1:                    0                     16                   15
    ## 2:                    0                     16                   15
    ## 3:                    0                     16                   15
    ## 4:                    0                     16                   15
    ## 5:                    0                     16                   15
    ## 6:                    0                     16                   15
    ##    ind_ctma_fin_ult1_01 ind_ctma_fin_ult1_10 ind_ctma_fin_ult1_11
    ## 1:                    0                    0                    0
    ## 2:                    0                    0                    0
    ## 3:                    0                    0                    0
    ## 4:                    0                    0                    0
    ## 5:                    0                    0                    0
    ## 6:                    0                    0                    0
    ##    ind_ctma_fin_ult1_0len ind_ctop_fin_ult1_00 ind_ctop_fin_ult1_01
    ## 1:                     16                   15                    0
    ## 2:                     16                   15                    0
    ## 3:                     16                   15                    0
    ## 4:                     16                   15                    0
    ## 5:                     16                   15                    0
    ## 6:                     16                   15                    0
    ##    ind_ctop_fin_ult1_10 ind_ctop_fin_ult1_11 ind_ctop_fin_ult1_0len
    ## 1:                    0                    0                     16
    ## 2:                    0                    0                     16
    ## 3:                    0                    0                     16
    ## 4:                    0                    0                     16
    ## 5:                    0                    0                     16
    ## 6:                    0                    0                     16
    ##    ind_ctpp_fin_ult1_00 ind_ctpp_fin_ult1_01 ind_ctpp_fin_ult1_10
    ## 1:                    0                    0                    0
    ## 2:                    0                    0                    0
    ## 3:                   15                    0                    0
    ## 4:                   15                    0                    0
    ## 5:                   15                    0                    0
    ## 6:                   15                    0                    0
    ##    ind_ctpp_fin_ult1_11 ind_ctpp_fin_ult1_0len ind_dela_fin_ult1_00
    ## 1:                   15                      0                   15
    ## 2:                   15                      0                   15
    ## 3:                    0                     16                    1
    ## 4:                    0                     16                   13
    ## 5:                    0                     16                   13
    ## 6:                    0                     16                    5
    ##    ind_dela_fin_ult1_01 ind_dela_fin_ult1_10 ind_dela_fin_ult1_11
    ## 1:                    0                    0                    0
    ## 2:                    0                    0                    0
    ## 3:                    1                    1                   12
    ## 4:                    0                    1                    1
    ## 5:                    0                    1                    1
    ## 6:                    0                    1                    9
    ##    ind_dela_fin_ult1_0len ind_ecue_fin_ult1_00 ind_ecue_fin_ult1_01
    ## 1:                     16                   15                    0
    ## 2:                     16                    0                    0
    ## 3:                      0                    0                    0
    ## 4:                     14                   15                    0
    ## 5:                     14                    0                    0
    ## 6:                      6                    0                    0
    ##    ind_ecue_fin_ult1_10 ind_ecue_fin_ult1_11 ind_ecue_fin_ult1_0len
    ## 1:                    0                    0                     16
    ## 2:                    0                   15                      0
    ## 3:                    0                   15                      0
    ## 4:                    0                    0                     16
    ## 5:                    0                   15                      0
    ## 6:                    0                   15                      0
    ##    ind_fond_fin_ult1_00 ind_fond_fin_ult1_01 ind_fond_fin_ult1_10
    ## 1:                   15                    0                    0
    ## 2:                   15                    0                    0
    ## 3:                   15                    0                    0
    ## 4:                   15                    0                    0
    ## 5:                   15                    0                    0
    ## 6:                   15                    0                    0
    ##    ind_fond_fin_ult1_11 ind_fond_fin_ult1_0len ind_hip_fin_ult1_00
    ## 1:                    0                     16                  15
    ## 2:                    0                     16                  15
    ## 3:                    0                     16                  15
    ## 4:                    0                     16                  15
    ## 5:                    0                     16                  15
    ## 6:                    0                     16                  15
    ##    ind_hip_fin_ult1_01 ind_hip_fin_ult1_10 ind_hip_fin_ult1_11
    ## 1:                   0                   0                   0
    ## 2:                   0                   0                   0
    ## 3:                   0                   0                   0
    ## 4:                   0                   0                   0
    ## 5:                   0                   0                   0
    ## 6:                   0                   0                   0
    ##    ind_hip_fin_ult1_0len ind_plan_fin_ult1_00 ind_plan_fin_ult1_01
    ## 1:                    16                   15                    0
    ## 2:                    16                    0                    0
    ## 3:                    16                   15                    0
    ## 4:                    16                   15                    0
    ## 5:                    16                   15                    0
    ## 6:                    16                    0                    0
    ##    ind_plan_fin_ult1_10 ind_plan_fin_ult1_11 ind_plan_fin_ult1_0len
    ## 1:                    0                    0                     16
    ## 2:                    0                   15                      0
    ## 3:                    0                    0                     16
    ## 4:                    0                    0                     16
    ## 5:                    0                    0                     16
    ## 6:                    0                   15                      0
    ##    ind_pres_fin_ult1_00 ind_pres_fin_ult1_01 ind_pres_fin_ult1_10
    ## 1:                   15                    0                    0
    ## 2:                   15                    0                    0
    ## 3:                   15                    0                    0
    ## 4:                   15                    0                    0
    ## 5:                   15                    0                    0
    ## 6:                   15                    0                    0
    ##    ind_pres_fin_ult1_11 ind_pres_fin_ult1_0len ind_reca_fin_ult1_00
    ## 1:                    0                     16                   15
    ## 2:                    0                     16                   15
    ## 3:                    0                     16                    0
    ## 4:                    0                     16                   15
    ## 5:                    0                     16                    0
    ## 6:                    0                     16                    0
    ##    ind_reca_fin_ult1_01 ind_reca_fin_ult1_10 ind_reca_fin_ult1_11
    ## 1:                    0                    0                    0
    ## 2:                    0                    0                    0
    ## 3:                    0                    0                   15
    ## 4:                    0                    0                    0
    ## 5:                    0                    0                   15
    ## 6:                    0                    0                   15
    ##    ind_reca_fin_ult1_0len ind_tjcr_fin_ult1_00 ind_tjcr_fin_ult1_01
    ## 1:                     16                    5                    3
    ## 2:                     16                    0                    0
    ## 3:                      0                    0                    0
    ## 4:                     16                   15                    0
    ## 5:                      0                    0                    0
    ## 6:                      0                    0                    0
    ##    ind_tjcr_fin_ult1_10 ind_tjcr_fin_ult1_11 ind_tjcr_fin_ult1_0len
    ## 1:                    4                    3                      1
    ## 2:                    0                   15                      0
    ## 3:                    0                   15                      0
    ## 4:                    0                    0                     16
    ## 5:                    0                   15                      0
    ## 6:                    0                   15                      0
    ##    ind_valo_fin_ult1_00 ind_valo_fin_ult1_01 ind_valo_fin_ult1_10
    ## 1:                    0                    0                    0
    ## 2:                   15                    0                    0
    ## 3:                    0                    0                    0
    ## 4:                    0                    0                    0
    ## 5:                    0                    0                    0
    ## 6:                    0                    0                    0
    ##    ind_valo_fin_ult1_11 ind_valo_fin_ult1_0len ind_viv_fin_ult1_00
    ## 1:                   15                      0                  15
    ## 2:                    0                     16                  15
    ## 3:                   15                      0                  15
    ## 4:                   15                      0                  15
    ## 5:                   15                      0                  15
    ## 6:                   15                      0                  15
    ##    ind_viv_fin_ult1_01 ind_viv_fin_ult1_10 ind_viv_fin_ult1_11
    ## 1:                   0                   0                   0
    ## 2:                   0                   0                   0
    ## 3:                   0                   0                   0
    ## 4:                   0                   0                   0
    ## 5:                   0                   0                   0
    ## 6:                   0                   0                   0
    ##    ind_viv_fin_ult1_0len ind_nomina_ult1_00 ind_nomina_ult1_01
    ## 1:                    16                 15                  0
    ## 2:                    16                  0                  0
    ## 3:                    16                 15                  0
    ## 4:                    16                 15                  0
    ## 5:                    16                  0                  0
    ## 6:                    16                 15                  0
    ##    ind_nomina_ult1_10 ind_nomina_ult1_11 ind_nomina_ult1_0len
    ## 1:                  0                  0                   16
    ## 2:                  0                 15                    0
    ## 3:                  0                  0                   16
    ## 4:                  0                  0                   16
    ## 5:                  0                 15                    0
    ## 6:                  0                  0                   16
    ##    ind_nom_pens_ult1_00 ind_nom_pens_ult1_01 ind_nom_pens_ult1_10
    ## 1:                   15                    0                    0
    ## 2:                    0                    0                    0
    ## 3:                   15                    0                    0
    ## 4:                   15                    0                    0
    ## 5:                    0                    0                    0
    ## 6:                   15                    0                    0
    ##    ind_nom_pens_ult1_11 ind_nom_pens_ult1_0len ind_recibo_ult1_00
    ## 1:                    0                     16                 15
    ## 2:                   15                      0                  0
    ## 3:                    0                     16                  0
    ## 4:                    0                     16                 15
    ## 5:                   15                      0                  0
    ## 6:                    0                     16                  0
    ##    ind_recibo_ult1_01 ind_recibo_ult1_10 ind_recibo_ult1_11
    ## 1:                  0                  0                  0
    ## 2:                  0                  0                 15
    ## 3:                  0                  0                 15
    ## 4:                  0                  0                  0
    ## 5:                  0                  0                 15
    ## 6:                  0                  0                 15
    ##    ind_recibo_ult1_0len
    ## 1:                   16
    ## 2:                    0
    ## 3:                    0
    ## 4:                   16
    ## 5:                    0
    ## 6:                    0

2\_xgboost
----------

``` r
#選擇要進行預測的20項商品，為了怕跑太久，所以先跑其中一個作範例
#product.list <- colnames(fread("C:/Users/aa006/Desktop/3rd Place Soulution/train_ver2.csv", select=25:48, nrows=0))[-c(1,2,10,11)]

product.list1 <- colnames(fread("C:/Users/aa006/Desktop/3rd Place Soulution/train_ver2.csv", select=25:48, nrows=0))[-c(1,2,10,11)]
product.list <-product.list1[1]

for(product in product.list) {
  
  #print(product)
#"ind_cco_fin_ult1"只抓"2015-12-28"，"ind_reca_fin_ult1"只抓 "2015-06-28"剩下抓 "2016-05-28"。
  
  if(product == "ind_cco_fin_ult1") {
    train.date <- "2015-12-28"
  } else if(product == "ind_reca_fin_ult1") {
    train.date <- "2015-06-28"
  } else {
    train.date <- "2016-05-28"
  }
  
  data.1 <- fread(paste0("C:/Users/aa006/Desktop/3rd Place Soulution/train_", train.date, ".csv"), colClasses=c(indrel_1mes="character", conyuemp="character"))
  data.2 <- fread(paste0("C:/Users/aa006/Desktop/3rd Place Soulution/count_", train.date, ".csv"))
  data.train <- cbind(data.1, data.2)
  
#當train.date= "2016-05-28"時，用2016-04-28作為驗證(val)，剩下的都用2016-05-28驗證(val)，2016-06-28測試(test)。  
  
  if(train.date == "2016-05-28") {
    data.1 <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/train_2016-04-28.csv", colClasses=c(indrel_1mes="character", conyuemp="character"))
    data.2 <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/count_2016-04-28.csv")
  } else {
    data.1 <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/train_2016-05-28.csv", colClasses=c(indrel_1mes="character", conyuemp="character"))
    data.2 <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/count_2016-05-28.csv")
  }
  data.val <- cbind(data.1, data.2)
  
  data.1 <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/test_2016-06-28.csv", colClasses=c(indrel_1mes="character", conyuemp="character"))
  data.2 <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/count_2016-06-28.csv")
  data.test <- cbind(data.1, data.2)
  
  rm(data.1)
  rm(data.2)
  gc()
  
#取出前一個月沒有購買的該產品的客戶。
  
  data.train <- data.train[data.train[[paste0(product,"_last")]] == 0]
  data.val <- data.val[data.val[[paste0(product,"_last")]] == 0]
  data.test <- data.test[data.test[[paste0(product,"_last")]] == 0]
 
#ind_actividad_cliente_from_to，將前一個月與當月的ind_actividad_cliente(是否活躍)合在一起當變數(例如前一個月1，當月1 變成11)。  
   
  data.train$ind_actividad_cliente_from_to <- paste(data.train$ind_actividad_cliente_last, data.train$ind_actividad_cliente)
  data.val$ind_actividad_cliente_from_to <- paste(data.val$ind_actividad_cliente_last, data.val$ind_actividad_cliente)
  data.test$ind_actividad_cliente_from_to <- paste(data.test$ind_actividad_cliente_last, data.test$ind_actividad_cliente)
  
  data.train$tiprel_1mes_from_to <- paste(data.train$tiprel_1mes_last, data.train$tiprel_1mes)
  data.val$tiprel_1mes_from_to <- paste(data.val$tiprel_1mes_last, data.val$tiprel_1mes)
  data.test$tiprel_1mes_from_to <- paste(data.test$tiprel_1mes_last, data.test$tiprel_1mes)

#把前一個月的欄位拿掉。 
  
  data.train[, ind_actividad_cliente_last:=NULL]
  data.val[, ind_actividad_cliente_last:=NULL]
  data.test[, ind_actividad_cliente_last:=NULL]
  
  data.train[, tiprel_1mes_last:=NULL]
  data.val[, tiprel_1mes_last:=NULL]
  data.test[, tiprel_1mes_last:=NULL]
  
#n_products_last 找出前一個月20項商品的持有狀況，利用apply進行橫排(每一客戶)加總，但test欄位數有差異所以位置不一樣!!
  
  data.train$n_products_last <- apply(data.train[, (1:20)+20+20, with=FALSE], 1, sum)
  data.val$n_products_last <- apply(data.val[, (1:20)+20+20, with=FALSE], 1, sum)
  data.test$n_products_last <- apply(data.test[, (1:20)+20, with=FALSE], 1, sum)
  
#products_last先產生一個空欄位，再將前一個月的20個商品串成一串 例如(10000010000000010000)。
  
  data.train$products_last <- ""
  data.val$products_last <- ""
  data.test$products_last <- ""
  for(j in 1:20) {
    data.train$products_last <- paste0(data.train$products_last, data.train[[j+20+20]])
    data.val$products_last <- paste0(data.val$products_last, data.val[[j+20+20]])
    data.test$products_last <- paste0(data.test$products_last, data.test[[j+20]])
  }
  
# exp.var去除年月、客代欄位，數值型變數不處理，類別型的變數根據二元與名目有不同的處理方式。
# 將變數為character挑出來，進行factor的轉換，如果是2分類的在第一個迴圈，將train、val、test先合再一起做運算，levels[1]<-0，levels[2]<-1，算完再分開來。
  
  exp.var <- colnames(data.test)[-(1:2)]
  for(var in exp.var) {
    if(class(data.train[[var]])=="character") {
      levels <- levels(as.factor(data.train[[var]]))
      if(length(levels) == 2) {
        temp <- c(data.train[[var]], data.val[[var]], data.test[[var]])
        temp <- ifelse(temp==levels[1], 0, ifelse(temp==levels[2], 1, NA))
        data.train[[var]] <- temp[1:nrow(data.train)]
        data.val[[var]] <- temp[(nrow(data.train)+1):(nrow(data.train)+nrow(data.val))]
        data.test[[var]] <- temp[(nrow(data.train)+nrow(data.val)+1):length(temp)]
      } else {
        
# replace with target mean作者的說明是說用目標平均去替換覺得比 index of the lexicographic order(字典序)的效果來的好
# 讀入train的多類別變數，並將本次迴圈的Y商品命名為target，利用此變數將多類別的變數進行轉換，利用目標在每個類別下有1無0的去計算在每個類別下的平均。
# 再將VAL和test裡面相同變數也進行match
        
        data.temp <- data.train[, c(var, product), with=FALSE]
        colnames(data.temp)[ncol(data.temp)] <- "target"
        target.mean <- data.temp[, .(target=mean(target)), by=var]
        data.val[[var]] <- target.mean$target[match(data.val[[var]], target.mean[[var]])]
        data.test[[var]] <- target.mean$target[match(data.test[[var]], target.mean[[var]])]
        
#產生一個NA列，只抽原本的後面數間隔4個數取1/4的資料去計算target的平均，對應到從前面數的1/4的類別值塞入平均值
        
        temp <- rep(NA, nrow(data.train))
        for(j in 1:4) {
          ids.1 <- -seq(j, nrow(data.train), by=4)
          ids.2 <- seq(j, nrow(data.train), by=4)
          target.mean <- data.temp[ids.1, .(target=mean(target)), by=var]
          temp[ids.2] <- target.mean$target[match(data.train[[var]][ids.2], target.mean[[var]])]
        }
        data.train[[var]] <- temp
      }
    }
  }
  rm(data.temp)
  
  #分別把X變數與Y變數取出來，單個變數做xgb，分別產生dtrain、dval、dtest(Y都是na)三個DMatrix，並且將data.val、data.test只留下客代。
  #DMatrix是XGB裡面自定義的資料矩陣類型，會在train開始時進行一遍預處理，來提高每次迭代的效率。
  
  x.train <- data.train[, exp.var, with=FALSE]
  y.train <- data.train[[product]]
  dtrain <- xgb.DMatrix(data=as.matrix(x.train), label=y.train)
  rm(data.train)
  rm(x.train)
  
  x.val <- data.val[, exp.var, with=FALSE]
  y.val <- data.val[[product]]
  dval <- xgb.DMatrix(data=as.matrix(x.val), label=y.val)
  data.val <- data.val[, .(ncodpers)]
  rm(x.val)
  
  x.test <- data.test[, exp.var, with=FALSE]
  dtest <- xgb.DMatrix(data=as.matrix(x.test), label=rep(NA, nrow(data.test)))
  data.test <- data.test[, .(ncodpers)]
  rm(x.test)
  
  gc()
  
# P.S
# nrounds： 幾棵樹
# etal： 預測為0.3，介於0 to 1，step size shrinkage(步長收縮)用于防止過度配適，在每一步boosting step後，可以直接獲得新features的權重，在縮小特徵權重可以讓boosting更保守，越小表示越robust。
# max_depth：預設為6，介於1 to 無限大，用來指定樹的最大深度。
# min_child_weight：預設為1，用來指定子節點的minimum sum of instance weight(hessian)，在樹分支的過程，葉節點的sum of instance weight小於此參數，就會放棄此分支，在線性回歸模型上，指的就是每個節點所需的最小個數，，數字越大表示越保守。
# objective：目標函數，在此選擇"binary:logistic"邏輯回歸損失函數進行二元分類。
# eval_metric：設定驗證資料的估計指標，會根據objective進行預設，這裡使用AUC (area under curve score 曲線下面積值)。  
# base_score： 全域 bias需要指定全部instances的起始預測分數。
  
  nrounds <- 1000
  early_stopping_round <- 50
  params <- list("eta"=0.05,
                 "max_depth"=4,
                 "min_child_weight"=1,
                 "objective"="binary:logistic",
                 "eval_metric"="auc")
  
  set.seed(0)
  model.xgb <- xgb.train(params=params,
                         data=dtrain,
                         nrounds=nrounds,
                         watchlist=list(train=dtrain, val=dval),
                         early_stopping_round=early_stopping_round,
                         print_every_n=10,
                         base_score=mean(y.train))
  #每個迴圈會將每個產品的預測值計算出來並匯出CSV
  result <- data.table(data.val$ncodpers, predict(model.xgb, dval))
  colnames(result) <- c("ncodpers", product)
  result <- result[order(ncodpers)]
  #write.csv(result, paste0("validation_", product, "_", train.date, ".csv"), quote=FALSE, row.names=FALSE)
  
  result <- data.table(data.test$ncodpers, predict(model.xgb, dtest))
  colnames(result) <- c("ncodpers",product)
  result <- result[order(ncodpers)]
  #write.csv(result, paste0("submission_", product, "_", train.date, ".csv"), quote=FALSE, row.names=FALSE)
  
  rm(dtrain)
  rm(dval)
  rm(dtest)
  rm(result)
  gc()
  
  #save(model.xgb, file=paste0("xgboost_", product, "_", train.date, ".model"))
  
}
```

    ## 
    Read 48.7% of 904294 rows
    Read 87.4% of 904294 rows
    Read 904294 rows and 62 (of 62) columns from 0.170 GB file in 00:00:04
    ## 
    Read 33.2% of 904294 rows
    Read 58.6% of 904294 rows
    Read 80.7% of 904294 rows
    Read 904294 rows and 100 (of 100) columns from 0.191 GB file in 00:00:05
    ## 
    Read 36.7% of 926663 rows
    Read 65.8% of 926663 rows
    Read 926663 rows and 62 (of 62) columns from 0.174 GB file in 00:00:04
    ## 
    Read 25.9% of 926663 rows
    Read 50.7% of 926663 rows
    Read 72.3% of 926663 rows
    Read 93.9% of 926663 rows
    Read 926663 rows and 100 (of 100) columns from 0.199 GB file in 00:00:06
    ## 
    Read 68.8% of 929615 rows
    Read 929615 rows and 42 (of 42) columns from 0.139 GB file in 00:00:03
    ## 
    Read 28.0% of 929615 rows
    Read 50.6% of 929615 rows
    Read 74.2% of 929615 rows
    Read 97.9% of 929615 rows
    Read 929615 rows and 100 (of 100) columns from 0.203 GB file in 00:00:06
    ## [1]  train-auc:0.930297  val-auc:0.883715 
    ## Multiple eval metrics are present. Will use val_auc for early stopping.
    ## Will train until val_auc hasn't improved in 50 rounds.
    ## 
    ## [11] train-auc:0.972866  val-auc:0.954786 
    ## [21] train-auc:0.975078  val-auc:0.956631 
    ## [31] train-auc:0.976887  val-auc:0.957951 
    ## [41] train-auc:0.977525  val-auc:0.958331 
    ## [51] train-auc:0.977962  val-auc:0.958301 
    ## [61] train-auc:0.978802  val-auc:0.959679 
    ## [71] train-auc:0.979473  val-auc:0.960209 
    ## [81] train-auc:0.980299  val-auc:0.960764 
    ## [91] train-auc:0.980540  val-auc:0.961069 
    ## [101]    train-auc:0.980683  val-auc:0.961154 
    ## [111]    train-auc:0.980967  val-auc:0.961360 
    ## [121]    train-auc:0.981184  val-auc:0.961457 
    ## [131]    train-auc:0.981342  val-auc:0.961573 
    ## [141]    train-auc:0.981495  val-auc:0.961710 
    ## [151]    train-auc:0.981597  val-auc:0.961924 
    ## [161]    train-auc:0.981725  val-auc:0.962034 
    ## [171]    train-auc:0.981879  val-auc:0.962087 
    ## [181]    train-auc:0.982010  val-auc:0.962146 
    ## [191]    train-auc:0.982158  val-auc:0.962177 
    ## [201]    train-auc:0.982261  val-auc:0.962270 
    ## [211]    train-auc:0.982361  val-auc:0.962322 
    ## [221]    train-auc:0.982462  val-auc:0.962424 
    ## [231]    train-auc:0.982545  val-auc:0.962390 
    ## [241]    train-auc:0.982599  val-auc:0.962429 
    ## [251]    train-auc:0.982661  val-auc:0.962451 
    ## [261]    train-auc:0.982770  val-auc:0.962515 
    ## [271]    train-auc:0.982816  val-auc:0.962538 
    ## [281]    train-auc:0.982896  val-auc:0.962536 
    ## [291]    train-auc:0.982986  val-auc:0.962531 
    ## [301]    train-auc:0.983034  val-auc:0.962586 
    ## [311]    train-auc:0.983079  val-auc:0.962615 
    ## [321]    train-auc:0.983177  val-auc:0.962636 
    ## [331]    train-auc:0.983222  val-auc:0.962675 
    ## [341]    train-auc:0.983287  val-auc:0.962734 
    ## [351]    train-auc:0.983358  val-auc:0.962775 
    ## [361]    train-auc:0.983430  val-auc:0.962754 
    ## [371]    train-auc:0.983509  val-auc:0.962745 
    ## [381]    train-auc:0.983556  val-auc:0.962747 
    ## [391]    train-auc:0.983601  val-auc:0.962726 
    ## [401]    train-auc:0.983662  val-auc:0.962730 
    ## Stopping. Best iteration:
    ## [359]    train-auc:0.983401  val-auc:0.962801

P.S xgb的參數設定 可參考[XGB參數](https://www.analyticsvidhya.com/blog/2016/01/xgboost-algorithm-easy-steps/) ，[XGB參數2](https://github.com/dmlc/xgboost/blob/master/doc/parameter.md)。

``` r
#單樣商品的預測結果。
data4 <-fread("C:/Users/aa006/Desktop/3rd Place Soulution/submission_ind_cder_fin_ult1_2016-05-28.csv", header = T)
head(data4)
```

    ##    ncodpers ind_cder_fin_ult1
    ## 1:    15889      1.112727e-06
    ## 2:    15890      3.025012e-05
    ## 3:    15892      1.199947e-05
    ## 4:    15893      1.112727e-06
    ## 5:    15894      1.199947e-05
    ## 6:    15895      1.199947e-05

3\_make\_submission
-------------------

``` r
mode <- "submission"
product.list <- colnames(fread("C:/Users/aa006/Desktop/3rd Place Soulution/train_ver2.csv", select=25:48, nrows=0))
date.list <- "2016-05-28"

submission <- fread("C:/Users/aa006/Desktop/3rd Place Soulution/sample_submission.csv")[, .(ncodpers)]

result <- data.table()
for(i in 1:length(date.list)) {
 # print(date.list[i])
  #將每個客戶的每個商品的預測值整理在一起
  result.temp <- data.table()
  for(j in 1:length(product.list)) {
    product <- product.list[j]
    
    if(j %in% c(1,2,10,11)) {
      temp <- submission
      temp$pr  <- 1e-10
    } else {
      if(product == "ind_cco_fin_ult1") {
        train.date <- "2015-12-28"
      } else if(product == "ind_reca_fin_ult1") {
        train.date <- "2015-06-28"
      } else {
        train.date <- "2016-05-28"
      }
      temp <- fread(paste0("C:/Users/aa006/Desktop/3rd Place Soulution/submission_", product, "_", train.date, ".csv"))
      colnames(temp)[2] <- "pr"
      temp$product <- product
    }
    temp$product <- product
    result.temp <- rbind(result.temp, temp)
  }
  #-將每個客戶排除掉不同預測時間區間的兩個商品後將所有商品預測機率加總。
  #將每個客戶的每項商品除以自己加總的機率值做標準化，N作為有預測值的月份數
  # normalize
  pred.sum <- result.temp[product %in% product.list[-c(3,18)], .(sum=sum(pr)), by=ncodpers]
  result.temp <- merge(result.temp, pred.sum, by="ncodpers")
  result.temp <- result.temp[, .(ncodpers, log_pr=log(pr/sum), product)]
  result <- rbind(result, result.temp[, .(ncodpers, product, log_pr, N=1)])
  result <- result[, .(log_pr=sum(log_pr), N=sum(N)), by=.(ncodpers, product)]
}

# log-average 如果有預測多個月份的值因為加總時同一項商品有多個分數，所以要除以月份數。
result$log_pr <- result$log_pr / result$N
#--------------------------
#對分數進行由大到小的排序後跑一個迴圈，找出前七個商品，先抓一個客代的第一項商品，塞入submission，再把第一商品排掉。
# elect top 7 products
result <- result[order(ncodpers, -log_pr)]
for(i in 1:7) {
  #print(i)
  temp <- result[!duplicated(result, by="ncodpers"), .(ncodpers, product)]
  submission <- merge(submission, temp, by="ncodpers", all.x=TRUE)
  result <- result[duplicated(result, by="ncodpers"), .(ncodpers, product)]
  colnames(submission)[ncol(submission)] <- paste0("p",i)
  if(nrow(result) == 0) {
    break
  }
}
#上面是客代加七個商品欄位，現在把欄位名稱改成added_products，並且把商品名稱都黏在一起
submission[is.na(submission)] <- ""
submission$added_products <- submission[[paste0("p",1)]]
for(i in 2:7) {
  submission$added_products <- paste(submission$added_products, submission[[paste0("p",i)]])
}

#submission <- submission[order(ncodpers)]
#file.name <- paste0(mode, ".csv")
#write.csv(submission[, .(ncodpers, added_products)], file.name, quote=FALSE, row.names=FALSE)

data5 <-fread("C:/Users/aa006/Desktop/3rd Place Soulution/submission.csv", header = T)
head(data5)
```

    ##    ncodpers
    ## 1:    15889
    ## 2:    15890
    ## 3:    15892
    ## 4:    15893
    ## 5:    15894
    ## 6:    15895
    ##                                                                                                                  added_products
    ## 1:     ind_recibo_ult1 ind_reca_fin_ult1 ind_ecue_fin_ult1 ind_nomina_ult1 ind_nom_pens_ult1 ind_cno_fin_ult1 ind_ctop_fin_ult1
    ## 2: ind_reca_fin_ult1 ind_cco_fin_ult1 ind_valo_fin_ult1 ind_ctop_fin_ult1 ind_fond_fin_ult1 ind_pres_fin_ult1 ind_dela_fin_ult1
    ## 3:    ind_nom_pens_ult1 ind_nomina_ult1 ind_cno_fin_ult1 ind_ctpp_fin_ult1 ind_ctop_fin_ult1 ind_fond_fin_ult1 ind_viv_fin_ult1
    ## 4:   ind_cco_fin_ult1 ind_ecue_fin_ult1 ind_tjcr_fin_ult1 ind_recibo_ult1 ind_nom_pens_ult1 ind_reca_fin_ult1 ind_ctop_fin_ult1
    ## 5:   ind_cno_fin_ult1 ind_ctpp_fin_ult1 ind_ctop_fin_ult1 ind_hip_fin_ult1 ind_fond_fin_ult1 ind_viv_fin_ult1 ind_plan_fin_ult1
    ## 6:    ind_nomina_ult1 ind_nom_pens_ult1 ind_cno_fin_ult1 ind_dela_fin_ult1 ind_ctpp_fin_ult1 ind_ctop_fin_ult1 ind_viv_fin_ult1
