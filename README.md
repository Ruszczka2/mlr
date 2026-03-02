# MLR - Przewidywanie czasu trwania podcastu

**Autorzy:** Jan Tlałka, Radosław Ruczka

## Wstęp do zagadnienia

Poniższa praca skupia się na zagadnieniu przewidywania długości odsłuchu podcastu na podstawie jego cech. Starano się stworzyć model możliwych do wykorzystania przed lub w trakcie tworzenia podcastu. Z tego powodu starano się wykorzystać jedynie cechy, które są znane bądź możliwe do modyfikacji przez prowadzącego/twórcę podcastu.

Dane pochodzą z konkursu na Kaggle: [Playground Series S5E4](https://www.kaggle.com/competitions/playground-series-s5e4/overview)

## Opis danych

### Dane wejściowe (features)

- **id** – Unikalny identyfikator dla każdego odcinka
- **Podcast_Name** – Nazwa podcastu
- **Episode_Title** – Tytuł odcinka podcastu
- **Episode_Length_minutes** – Czas trwania odcinka podcastu
- **Genre** – Kategoria podcastu (np. Edukacja, Zdrowie)
- **Host_Popularity_percentage** – Popularność prowadzącego podcastu (0–100%)
- **Publication_Day** – Dzień tygodnia publikacji odcinka
- **Publication_Time** – Pora dnia publikacji (Morning, Afternoon, Evening, Night)
- **Guest_Popularity_percentage** – Popularność gościa (0–100%) lub brak
- **Number_of_Ads** – Liczba reklam w odcinku
- **Episode_Sentiment** – Opinia o podcaście (Negative, Neutral, Positive)

### Dane wyjściowe (target)

- **Listening_Time_minutes** – Rzeczywista długość odsłuchu przez słuchaczy

## Etapy projektu

### 1. Załadowanie i eksploracja danych

- Załadowanie danych z pliku `train.csv`
- Analiza statystyczna (describe, unique values)
- Wstępna wizualizacja histogramów i wykresów rozrzutu

### 2. Czyszczenie danych

Kroki czyszczenia obejmowały:

- Usunięcie danych odstających (wejściowych):
  - `Number_of_Ads < 4`
  - `Episode_Length_minutes < 140`
  - `Host_Popularity_percentage` między 20% a 100%
  - `Guest_Popularity_percentage ≤ 100%`

- Usunięcie danych odstających (wyjściowych):
  - `Listening_Time_minutes ≤ Episode_Length_minutes`

- Uzupełnienie brakujących wartości:
  - Kolumna `Guest_Popularity_percentage`: zastąpienie medianą (okazało się najlepszym rozwiązaniem)

**Wynik:** ~90% danych zachowanych po czyszczeniu

### 3. Transformacja danych

Zmiana danych kategorycznych na liczbowe:
- `Episode_Sentiment`: Negative → -1, Neutral → 0, Positive → 1
- `Publication_Time`: Morning → 0, Afternoon → 1, Evening → 2, Night → 3
- `Publication_Day`: Monday → 0, ..., Sunday → 6
- `Genre`: kodowanie numeryczne
- `Episode_Title`: ekstrakcja numeru odcinka (string → int)

### 4. Analiza korelacji

Obliczono macierz korelacji Spearmana. Najistotniejsze cechy:
- **Episode_Length_minutes** – bardzo silna korelacja z wynikiem
- **Number_of_Ads** – słaba korelacja
- **Host_Popularity_percentage** – słaba korelacja

### 5. Podział danych

- **Dane treningowe:** 80% zbioru
- **Dane testowe:** 20% zbioru

Wybrane cechy do modelowania:
- `Episode_Length_minutes`
- `Number_of_Ads`
- `Host_Popularity_percentage`

## Testy modeli

Aby ocenić modele użyto dwóch metryk (w minutach):

$$RMSE = \sqrt{\frac{\sum_{i=1}^{N}(y_i - y_{i\_pred})^2}{N}}$$

$$MAE = \frac{\sum_{i=1}^{N}|y_i - y_{i\_pred}|}{N}$$

### Klasyczne metody

#### Regresja liniowa
- Najprostsza metoda przewidywania
- Szybka, ale niekoniecznie dająca małe błędy
- **Wynik na danych testowych:** RMSE ≈ 10.99, MAE ≈ 8.58

#### Drzewo decyzyjne
- Ideowo prosta metoda z tendencją do przeuczenia
- Głębokość: max_depth=7 (dobrana metodą prób i błędów)
- **Wynik na danych testowych:** RMSE ≈ 10.73, MAE ≈ 8.34

#### Las losowy
- Rozszerzenie drzewa decyzyjnego metodą baggingu
- Wiele drzew → lepsze wyniki
- Głębokość: max_depth=7
- **Wynik na danych testowych:** RMSE = 10.48, MAE = 8.08 ✓ **Najlepszy klasyczny**

#### Sieć neuronowa (MLP)
- Najbardziej zaawansowana metoda
- Architektura: 1 warstwa ukryta (30 neuronów), ReLU, Adam solver
- max_iter=50
- **Wynik na danych testowych:** RMSE ≈ 10.53, MAE ≈ 8.17

### Podejście szarej skrzynki (Gray Box)

Integracja wiedzy eksperckiej z metodami ML. Dwa podejścia:

#### Biała skrzynka (WhiteBox)

Model oparty na wiedzy ekspercką z dyskusji Kaggle:

$$listening\_time = episode\_length \cdot (a_1 + a_2 \cdot number\_ads + a_3 \cdot Host\_popularity)$$

Gdzie $a_1, a_2, a_3$ to współczynniki dobierane metodą regresji liniowej.

Wejścia do modelu:
- `Episode_Length_minutes`
- `Lenght_X_Host` (iloczyn długości i popularności)
- `Lenght_X_Ads` (iloczyn długości i liczby reklam)

**Wynik na danych testowych:** RMSE ≈ 10.66, MAE ≈ 8.26

#### Równoległe (Parallel)

Szara skrzynka (ML model) przewiduje korektę wyniku białej skrzynki.

- **Neural Network (Parallel):** RMSE ≈ 10.52, MAE ≈ 8.18
- **Random Forest (Parallel):** RMSE = 10.46, MAE = 8.07 ✓ **Najlepsze ogółem**

#### Szeregowe (Serial)

Wynik białej skrzynki dodawany jako wejście do czarnej skrzynki.

- **Neural Network (Serial):** RMSE ≈ 10.61, MAE ≈ 8.20
- **Random Forest (Serial):** RMSE ≈ 10.49, MAE ≈ 8.08

## Wyniki

### Podsumowanie

Wszystkie wykorzystane metody dały zbliżone wyniki (~10.5 minut RMSE, ~8.1 minut MAE).

**Najlepszy model:** Równoległy gray-box z lasem losowym
- **RMSE = 10.46 minut**
- **MAE = 8.07 minut**

Jest to niewielka poprawa względem samego lasu losowego, biorąc pod uwagę dodatkowe skomplikowanie modelu.

### Wnioski

- Prosta wiedza ekspercka (model biała skrzynka) daje rozumne wyniki
- Połączenie z metodami ML (gray-box) daje marginalne ulepszenie
- Las losowy konsekwentnie bardziej efektywny niż sieć neuronowa
- Brak znaczącej korelacji innych cech z wynikiem ogranicza potencjał modelowania

## Wykorzystane narzędzia

- Python 3.8+
- pandas, numpy
- scikit-learn
- matplotlib
- datashader


## Źródła

- Dyskusja na Kaggle (wymiana doświadczeń o modelach): https://www.kaggle.com/competitions/playground-series-s5e4/discussion/573002
- Dokumentacja scikit-learn: https://scikit-learn.org/
- Dokumentacja datashader: https://datashader.holoviz.org/
