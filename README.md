Magyar törvények adatai, 1998-
================
Ferenci Tamás (<https://www.medstat.hu/>)

- [Háttér, célkitűzés](#háttér-célkitűzés)
- [A megvalósítás alapvonalai](#a-megvalósítás-alapvonalai)
- [Technikai részletek](#technikai-részletek)
- [Eredmények](#eredmények)

## Háttér, célkitűzés

A magyar Országgyűlés honlapján egy elég jó keresővel elérhetőek a
benyújtott törvények adatai, arra azonban nincsen mód, hogy ezt gépi
úton feldolgozható formában lekérjük. Márpedig ez fontos lenne, például,
ha valamilyen elemzést szeretnénk készíteni a törvények adataira
vonatkozóan, netán vizualizációt készítenénk ezekből stb. Éppen ezért
célul tűztem ki, hogy automatizáltan letöltsem az összes törvény
adatait, majd azokat egy további feldolgozásra alkalmas formában
közzétegyem.

## A megvalósítás alapvonalai

Az [Országgyűlés honlapján](https://www.parlament.hu/) elérhető az
összes tárgyalt anyag, parlamenti szóhasználattal „iromány”. Ezek jól
[lekérdezhetők](https://www.parlament.hu/web/guest/iromanyok-elozo-ciklusbeli-adatai),
parlamenti ciklusonként külön-külön. Megtehetnénk, hogy letöltjük az
összes ilyet, majd utána kiszűrjük belőle a törvényjavaslatokat (mivel
iromány sok egyéb is lehet, beszámoló, azonnali kérdés, határozati
javaslat és így tovább), de az irományok száma olyan nagy, hogy ez
rendkívül időigényes, ráadásul ha csak a törvények érdekelnek minket,
akkor jórészt felesleges is. Itt jön segítségünkre az, hogy a kereső
által használt URL nagyon szabályos formájú, így könnyen előállítható az
a keresés, amely csak a törvényjavaslatokra szűkít (illetve érdemes azt
is beállítani, hogy a típus legyen „lezárt”, hiszen csak ezekből
lehetett törvény – természetesen ezeknek sem mindegyikéből, mert lehet,
hogy nem szavazták meg). Ha az így kapott eredménylista találatain
megyünk végig, akkor célirányosan letölthetjük csak a törvényeket. Az
egyetlen nehézség, hogy az eredménylista valójában 100 tételenként meg
van bontva, de az erre vezető linkek könnyen kikereshetőek, így ez nagy
bajt nem jelent.

Ha megvan a törvényjavaslatok listája, akkor már könnyű dolgunk van,
mert a tetején lévő második táblázat tartalmazza mindig az adatokat,
ráadásul szinte tökéletesen egységes struktúrában. Ezeket letöltve, majd
összefűzve megkapjuk a szükséges adatbázist.

A weboldalak automatikus letöltését az `rvest`, az adatok feldolgozását
a `data.table` csomag segítségével végeztem, mindezt az [R statisztikai
környezet](https://www.youtube.com/@FerenciTamas/playlists?view=50&sort=dd&shelf_id=2)
alatt.

Ezzel a módszerrel az 1998-as parlamenti ciklus, és az azt követő
ciklusok adatait tudtam megszerezni. Az 1994-1998 ciklus fent van ugyan
a Parlement honlapján, de teljesen más (nem feldolgozható) formátumban,
az 1990-1994 ciklus pedig fent sincs az irományok között.

## Technikai részletek

Elsőként mind a 7, feldolgozható parlamenti ciklusra letöltjük az
adatokat, és elmentjük az eredményt:

``` r
for(ciklus in 36:42) {
  if(!file.exists(paste0("./adatok/ciklus", ciklus, ".rds"))) {
    linkek <- rvest::read_html_live(paste0(
      "https://www.parlament.hu/web/guest/iromanyok-elozo-ciklusbeli-adatai?p_p_id=hu_parlament_cms_pair_",
      "portlet_PairProxy_INSTANCE_9xd2Wc9jP4z8&p_p_lifecycle=1&p_p_state=normal&p_p_mode=view&_hu_parlament_",
      "cms_pair_portlet_PairProxy_INSTANCE_9xd2Wc9jP4z8_pairAction=http%3A%2F%2Fwww.parlament.hu%2Finternet",
      "%2Fcplsql%2Fogy_irom.irom_lekerd%3FP_TIP%3Dnull%26P_CKL%3D", ciklus,
      "%26P_PARAM%3DI%26P_FOTIP%3Dnull", "%26P_FOTIP%3DT%26P_ATIP%3Dz&p_auth=MVj7t9MD"))
    linkek <- rvest::html_attr(rvest::html_elements(linkek, "a"), "href")
    linkek <- linkek[grep("www.parlament.hu:443", linkek)]
    
    linkektv <- do.call(c, sapply(linkek, function(l) {
      temp <- rvest::html_attr(rvest::html_elements(rvest::read_html_live(l), "a"), "href")
      temp <- temp[grep("ogy_irom.irom_adat", temp)]
    }))
    
    saveRDS(rbindlist(lapply(1:length(linkektv), function(i) {
      temp <- rvest::html_table(rvest::read_html_live(linkektv[i]))
      if(length(temp)==0) data.table(Jelleg = NA) else setNames(data.table(t(temp[[2]][[2]])), temp[[2]][[1]])
    }), fill = TRUE, idcol = "Szam"), paste0("./adatok/ciklus", ciklus, ".rds"))
  }
}
```

Azért érdemes ezt a megoldást választani, mert így, ha valamelyik
lépésben hiba történik, akkor könnyebb újrakezdeni (persze elegánsabb
lett volna egy `tryCatch` kivételkezelést használni). Ha sikerült, akkor
töltsünk be mindent:

``` r
res <- lapply(36:42, function(ciklus) readRDS(paste0("./adatok/ciklus", ciklus, ".rds")))
```

Az érdemi résznél a letöltött táblázat hosszát azért kell ellenőrizni,
mert néha, ha az oldal mérete nagyon nagy, akkor nem azonnal töltődik
be, hanem előtte egy homokóra fut, de ezt az `rvest` nem tudja kezelni.
Ha ezeket lementjük csupa `NA`-val kitöltve (a `fill = TRUE` miatt ez
megvalósul), akkor később azonosíthatóak:

``` r
sapply(res, function(x) x[is.na(Jelleg)])
```

    ## [[1]]
    ## Empty data.table (0 rows and 18 cols): Szam,Típus,Főtípus,Jelleg,Benyújtva,Új/átdolgozott változat...
    ## 
    ## [[2]]
    ## Empty data.table (0 rows and 19 cols): Szam,Típus,Főtípus,Jelleg,Benyújtva,Új/átdolgozott változat...
    ## 
    ## [[3]]
    ## Empty data.table (0 rows and 20 cols): Szam,Típus,Főtípus,Jelleg,Benyújtva,Új/átdolgozott változat...
    ## 
    ## [[4]]
    ## Empty data.table (0 rows and 20 cols): Szam,Típus,Főtípus,Jelleg,Benyújtva,Új/átdolgozott változat...
    ## 
    ## [[5]]
    ##     Szam  Típus Főtípus Jelleg Benyújtva Átiktatott változat Régi irományszám
    ##    <int> <char>  <char> <char>    <char>              <char>           <char>
    ## 1:   129   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 2:   168   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 3:   339   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 4:   754   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 5:  1177   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ##    Állapot Tárgyalási mód Napirendi pont típusa Kihirdetés száma MK száma
    ##     <char>         <char>                <char>           <char>   <char>
    ## 1:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 2:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 3:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 4:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 5:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ##    Kihirdetés dátuma Megjegyzés Benyújtó(k) Irományszöveg Elfogadott cím
    ##               <char>     <char>      <char>        <char>         <char>
    ## 1:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 2:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 3:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 4:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 5:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 
    ## [[6]]
    ##     Szam  Típus Főtípus Jelleg Benyújtva Átiktatott változat Régi irományszám
    ##    <int> <char>  <char> <char>    <char>              <char>           <char>
    ## 1:    61   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 2:   369   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 3:   530   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 4:   828   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ##    Állapot Tárgyalási mód Napirendi pont típusa Kihirdetés száma MK száma
    ##     <char>         <char>                <char>           <char>   <char>
    ## 1:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 2:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 3:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 4:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ##    Kihirdetés dátuma Megjegyzés Benyújtó(k) Irományszöveg Elfogadott cím
    ##               <char>     <char>      <char>        <char>         <char>
    ## 1:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 2:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 3:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 4:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 
    ## [[7]]
    ##     Szam  Típus Főtípus Jelleg Benyújtva Átiktatott változat Régi irományszám
    ##    <int> <char>  <char> <char>    <char>              <char>           <char>
    ## 1:    87   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 2:   278   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ## 3:   413   <NA>    <NA>   <NA>      <NA>                <NA>             <NA>
    ##    Állapot Tárgyalási mód Napirendi pont típusa Kihirdetés száma MK száma
    ##     <char>         <char>                <char>           <char>   <char>
    ## 1:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 2:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ## 3:    <NA>           <NA>                  <NA>             <NA>     <NA>
    ##    Kihirdetés dátuma Megjegyzés Benyújtó(k) Irományszöveg Elfogadott cím
    ##               <char>     <char>      <char>        <char>         <char>
    ## 1:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 2:              <NA>       <NA>        <NA>          <NA>           <NA>
    ## 3:              <NA>       <NA>        <NA>          <NA>           <NA>

Ha pedig ez megvan, akkor kézzel javíthatjuk őket:

``` r
res[[5]] <- rbind(res[[5]][Szam!=129], setNames(data.table(
  129, "törvényjavaslat", "törvényjavaslat", "új", "2014.11.20.", "",
  "",  "kihirdetve", "normál", "Nemzetiségi", "XCIX", "184", "2014.12.29.",
  "", "kormány (nemzetgazdasági miniszter)", "szöveges PDF", ""), names(res[[5]])))

res[[5]] <- rbind(res[[5]][Szam!=168], setNames(data.table(
  168, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2014.10.30.", "",
  "",  "kihirdetve", "határozati házszabályi rendelkezésektől való eltéréssel", "NemzetiségiUniós", "C",
  "184", "2014.12.29.", "", "kormány (nemzetgazdasági miniszter)", "szöveges PDF", ""), names(res[[5]])))

res[[5]] <- rbind(res[[5]][Szam!=339], setNames(data.table(
  339, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2015.05.13.", "",
  "",  "kihirdetve", "határozati házszabályi rendelkezésektől való eltéréssel", "NemzetiségiUniós", "C",
  "97", "2015.07.03.",
  "A Kormány kezdeményezésére a 2015. június 16-17-i rendkívüli ülés napirendjén. A Kormány kezdeményezésére a 2015. június 22-23-i rendkívüli ülés napirendjén.",
  "kormány (nemzetgazdasági miniszter)", "HTML", ""), names(res[[5]])))

res[[5]] <- rbind(res[[5]][Szam!=754], setNames(data.table(
  754, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2016.04.26.", "",
  "",  "kihirdetve", "határozati házszabályi rendelkezésektől való eltéréssel", "NemzetiségiUniós", "XC",
  "91", "2016.06.24.", "",
  "kormány (nemzetgazdasági miniszter)", "HTML", ""), names(res[[5]])))

res[[5]] <- rbind(res[[5]][Szam!=1177], setNames(data.table(
  1177, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2017.05.02.", "",
  "",  "kihirdetve", "határozati házszabályi rendelkezésektől való eltéréssel", "NemzetiségiUniós", "C",
  "100", "2017.06.27.", "ParLexben feldolgozott iromány.",
  "kormány (nemzetgazdasági miniszter)", "HTML", ""), names(res[[5]])))

res[[6]] <- rbind(res[[6]][Szam!=61], setNames(data.table(
  61, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2018.06.13.", "",
  "",  "kihirdetve", "határozati házszabályi rendelkezésektől való eltéréssel", "NemzetiségiUniós", "L",
  "123", "2018.07.31.",
  "ParLexben feldolgozott iromány.A Kormány kezdeményezésére a 2018. június 25-29-i rendkívüli ülés napirendjén. A Kormány kezdeményezésére a 2018. július 16-17-i és július 20-i rendkívüli ülés napirendjén.",
  "kormány (pénzügyminiszter)", "HTML", ""), names(res[[6]])))

res[[6]] <- rbind(res[[6]][Szam!=369], setNames(data.table(
  369, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2019.06.04.", "",
  "",  "kihirdetve", "normál", "NemzetiségiUniós", "LXXI",
  "128", "2019.07.23.",
  "ParLexben feldolgozott iromány.A Kormány kezdeményezésére a 2019. június 17-21-i rendkívüli ülés napirendjén. A Kormány kezdeményezésére a 2019. július 8-i rendkívüli ülés napirendjén. A Kormány kezdeményezésére a 2019. július 12-i rendkívüli ülés napirendjén.",
  "kormány (pénzügyminiszter)", "HTML", ""), names(res[[6]])))

res[[6]] <- rbind(res[[6]][Szam!=530], setNames(data.table(
  530, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2020.05.26.", "",
  "",  "kihirdetve", "normál", "NemzetiségiUniós", "XC",
  "170", "2020.07.15.",
  "ParLexben feldolgozott iromány.A Kormány kezdeményezésére a 2020. június 29-július 3-i rendkívüli ülésszak napirendjén.",
  "kormány (pénzügyminiszter)", "HTML", ""), names(res[[6]])))

res[[6]] <- rbind(res[[6]][Szam!=828], setNames(data.table(
  828, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2021.05.04.", "",
  "",  "kihirdetve", "normál", "NemzetiségiUniós", "XC",
  "120", "2021.06.25.",
  "ParLexben feldolgozott iromány.",
  "kormány (pénzügyminiszter)", "HTML", ""), names(res[[6]])))

res[[7]] <- rbind(res[[7]][Szam!=87], setNames(data.table(
  87, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2022.06.07.", "",
  "",  "kihirdetve", "normál", "NemzetiségiUniós", "XXV", "127", "2022.07.27.",
  "IJR-ParLexben feldolgozott iromány. A törvényjavaslat szövegét az előterjesztő kérésére 2022.06.07-én 16:55 órakor lecseréltük. Az előterjesztő tájékoztató levele és a törvényjavaslat lecserélt szövege az Irományhoz kapcsolódó háttéranyagok és egyéb dokumentumok között megtalálható. A Kormány kezdeményezésére a 2022. június 20-24-i rendkívüli ülés napirendjén. A Kormány kezdeményezésére a 2022. július 11-12-i rendkívüli ülés napirendjén. A Kormány kezdeményezésére a 2022. július 18-19-i rendkívüli ülés napirendjén.",
  "kormány (pénzügyminiszter)", "HTML", ""), names(res[[7]])))

res[[7]] <- rbind(res[[7]][Szam!=278], setNames(data.table(
  278, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2023.05.30.", "",
  "",  "kihirdetve", "normál", "NemzetiségiUniós", "LV", "104", "2023.07.13.",
  "IJR-ParLexben benyújtott iromány. A Kormány kezdeményezésére a 2023. július 3-4-7-i rendkívüli ülés napirendjén.",
  "kormány (pénzügyminiszter)", "HTML", ""), names(res[[7]])))

res[[7]] <- rbind(res[[7]][Szam!=413], setNames(data.table(
  413, "költségvetési törvényjavaslat", "törvényjavaslat", "új", "2024.11.11.", "",
  "",  "kihirdetve", "normál", "Nemzetiségi", "XC", "136", "2024.12.30.",
  "IJR-ParLexben benyújtott iromány.A Kormány kezdeményezésére az Országgyűlés 2024. december 16-17-18-20-i rendkívüli ülésének napirendjén.",
  "kormány (pénzügyminiszter)", "HTML", ""), names(res[[7]])))
```

Ezt követően végrehajthatjuk az összefűzést:

``` r
res2 <- rbindlist(res, idcol = "Ciklus", fill = TRUE)
```

Célszerű a ciklusoknak adni egy jól értelmezhető nevet is:

``` r
res2 <- merge(res2, data.table(Ciklus = 1:7,
                               CiklusEv = c("1998-2002", "2002-2006", "2006-2010",
                                            "2010-2014", "2014-2018", "2018-2022", "2022-")))
```

Jelenleg minden változó szöveges, ezt transzformáljuk a valóságnak
megfelelően, például a dátumokat alakítsuk dátummá:

``` r
res2$Benyújtva <- as.Date(res2$Benyújtva, format = "%Y.%m.%d")
res2$`Kihirdetés dátuma` <- as.Date(res2$`Kihirdetés dátuma`, format = "%Y.%m.%d")
```

Ezek alapján kiszámolhatunk egyéb mutatókat is, például a benyújtástól a
kihirdetésig eltelt időt:

``` r
res2$Delay <- as.numeric(difftime(res2$`Kihirdetés dátuma`, res2$Benyújtva, units = "days"))
```

Végezetül mentsük ki feldolgozható formátumban az adatokat:

``` r
saveRDS(res2, "TorvenyAdatok.rds")
fwrite(res2, "TorvenyAdatok.csv", dec = ",", sep = ";", bom = TRUE)
```

## Eredmények

Az elkészült adatbázis [innen
letölthető](https://raw.githubusercontent.com/tamas-ferenci/TorvenyAdatok/main/TorvenyAdatok.csv),
további, gépi úton történő feldolgozásra, elemzésre, vizualizációra
alkalmas formátumban.
