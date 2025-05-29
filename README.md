# Analiza formatu PNG - Projekt E-media

## Jak działa format PNG?

### Struktura pliku PNG

Każdy plik PNG składa się z dokładnie określonych elementów w ustalonej kolejności:

```
[SYGNATURA - 8 bajtów]
[CHUNK 1]
[CHUNK 2]
[CHUNK 3]
...
[CHUNK IEND]
```

### Sygnatura PNG

Pierwsze 8 bajtów każdego pliku PNG to zawsze ta sama sekwencja:
```
89 50 4E 47 0D 0A 1A 0A
```

Każdy bajt ma swoje znaczenie:

**Bajt 1: `89` (137 dziesiętnie)**
- Wartość większa niż 127 (maksymalna wartość ASCII)
- Zapobiega interpretacji pliku PNG jako pliku tekstowego
- Jeśli program tekstowy próbuje otworzyć PNG, ten bajt sygnalizuje że to plik binarny

**Bajty 2-4: `50 4E 47` (80, 78, 71 dziesiętnie)**
- To litery "PNG" w kodowaniu ASCII
- P = 80 (0x50 hex)
- N = 78 (0x4E hex)  
- G = 71 (0x47 hex)
- Pozwalają łatwo zidentyfikować typ pliku

**Bajty 5-6: `0D 0A` (13, 10 dziesiętnie)**
- `0D` = Carriage Return (CR, \r) - powrót karetki
- `0A` = Line Feed (LF, \n) - nowa linia
- To sekwencja końca linii w systemie Windows (CRLF)
- Wykrywa niewłaściwą konwersję pliku między systemami (Unix używa tylko LF, Windows CRLF)

**Bajt 7: `1A` (26 dziesiętnie)**
- Znak SUB (Substitute) w ASCII, nazywany też Ctrl+Z
- W systemie DOS/Windows oznacza koniec pliku tekstowego
- Zatrzymuje wyświetlanie pliku w starych edytorach tekstu DOS

**Bajt 8: `0A` (10 dziesiętnie)**
- Line Feed (LF, \n) - znak nowej linii w systemach Unix/Linux
- Dodatkowe zabezpieczenie przed konwersją końców linii

### Struktura chunka

Chunk (segment danych) to podstawowa jednostka organizacji danych w PNG. Każdy chunk ma identyczną strukturę:

```
[DŁUGOŚĆ - 4 bajty] 
[TYP - 4 bajty]     
[DANE - n bajtów]   
[CRC - 4 bajty]     
```

**DŁUGOŚĆ (4 bajty)**
- Liczba 32-bitowa bez znaku w formacie big-endian
- Określa ile bajtów mają DANE (nie wlicza typu ani CRC)
- Maksymalna długość: 2^31-1 bajtów (około 2 GB)
- Przykład: `00 00 00 0D` = 13 bajtów danych

**TYP (4 bajty)**
- 4 znaki ASCII określające rodzaj chunka
- Wielkość liter ma znaczenie:
  - Pierwsza litera: wielka = chunk krytyczny, mała = dodatkowy
  - Druga litera: wielka = publiczny, mała = prywatny
  - Trzecia litera: zarezerwowana (musi być wielka)
  - Czwarta litera: wielka = można kopiować, mała = nie kopiować
- Przykład: `49 48 44 52` = "IHDR"

**DANE (długość określona w polu DŁUGOŚĆ)**
- Właściwa zawartość chunka
- Format zależy od typu chunka
- Może być pusta (długość = 0)

**CRC (4 bajty)**
- Cyclic Redundancy Check - suma kontrolna
- 32-bitowa wartość obliczona z pól TYP + DANE
- Algorytm CRC-32 z wielomianem 0xEDB88320
- Pozwala wykryć uszkodzenia danych

### Chunki krytyczne (obowiązkowe)

**1. IHDR (Image Header) - Nagłówek obrazu**

Zawsze pierwszy chunk po sygnaturze. Struktura (13 bajtów):

```
Bajty 0-3:  Szerokość (width) - 32 bity, big-endian
Bajty 4-7:  Wysokość (height) - 32 bity, big-endian  
Bajt 8:     Głębia bitowa (bit depth) - 1, 2, 4, 8 lub 16
Bajt 9:     Typ koloru (color type):
            0 = skala szarości
            2 = RGB (true color)
            3 = paleta kolorów (indexed)
            4 = skala szarości z kanałem alfa
            6 = RGBA (RGB z kanałem alfa)
Bajt 10:    Metoda kompresji - zawsze 0 (deflate/inflate)
Bajt 11:    Metoda filtrowania - zawsze 0
Bajt 12:    Metoda przeplotu (interlace):
            0 = brak przeplotu
            1 = przeplot Adam7
```

**2. PLTE (Palette) - Paleta kolorów**

Obowiązkowy tylko dla typu koloru 3. Struktura:
- Serie 3-bajtowych wpisów RGB
- Maksymalnie 256 kolorów (768 bajtów)
- Każdy wpis: [R][G][B] gdzie R,G,B to bajty 0-255

**3. IDAT (Image Data) - Dane obrazu**

- Zawiera skompresowane piksele obrazu
- Kompresja algorytmem deflate (jak w ZIP)
- Może być wiele chunków IDAT (dane podzielone)
- Przed kompresją stosowane są filtry PNG

**4. IEND (Image End) - Koniec obrazu**

- Zawsze ostatni chunk
- Długość danych = 0 (pusty)
- Sygnalizuje koniec pliku PNG

### Chunki dodatkowe (ancillary chunks)

**tEXt - Tekst nieskompresowany**
```
Słowo kluczowe (1-79 bajtów, ASCII)
Bajt zerowy (separator)
Tekst (ASCII Latin-1)
```

**tIME - Znacznik czasu**
```
Bajty 0-1: Rok (big-endian)
Bajt 2:    Miesiąc (1-12)
Bajt 3:    Dzień (1-31)
Bajt 4:    Godzina (0-23)
Bajt 5:    Minuta (0-59)
Bajt 6:    Sekunda (0-60, 60 dla sekund przestępnych)
```

**pHYs - Wymiary fizyczne**
```
Bajty 0-3: Piksele na jednostkę, oś X (big-endian)
Bajty 4-7: Piksele na jednostkę, oś Y (big-endian)
Bajt 8:    Specyfikator jednostki (0=nieznana, 1=metr)
```

**eXIf - Dane EXIF**
- Zawiera metadane w formacie EXIF
- Informacje o aparacie, ustawieniach, GPS
- Format zgodny ze standardem EXIF

**iTXt - Międzynarodowy tekst**
- Podobny do tEXt ale z obsługą Unicode (UTF-8)
- Może zawierać tekst w dowolnym języku

### Kolejność bajtów (Big-endian)

PNG używa zapisu big-endian dla liczb wielobajtowych. Oznacza to, że najbardziej znaczący bajt jest pierwszy.

Przykład dla liczby 1920 (0x780 hex):
```
32-bitowy zapis: 00 00 07 80
                 ^         ^
                 MSB      LSB
                 
MSB = Most Significant Byte (najbardziej znaczący)
LSB = Least Significant Byte (najmniej znaczący)
```

### Algorytm CRC-32

CRC w PNG używa wielomianu generującego: x^32 + x^26 + x^23 + x^22 + x^16 + x^12 + x^11 + x^10 + x^8 + x^7 + x^5 + x^4 + x^2 + x + 1

W zapisie szesnastkowym: 0xEDB88320

Proces obliczania:
1. Inicjalizacja rejestru CRC wartością 0xFFFFFFFF
2. Dla każdego bajtu danych:
   - XOR bajt z najmłodszym bajtem rejestru
   - Dla każdego z 8 bitów:
     - Jeśli najmłodszy bit = 1: przesuń w prawo i XOR z wielomianem
     - Jeśli najmłodszy bit = 0: tylko przesuń w prawo
3. XOR końcowa wartość z 0xFFFFFFFF

### Filtry PNG

Przed kompresją każda linia pikseli może być filtrowana. Typ filtru (1 bajt) poprzedza dane każdej linii:

- 0 = None - brak filtrowania
- 1 = Sub - różnica z pikselem po lewej
- 2 = Up - różnica z pikselem powyżej
- 3 = Average - średnia z lewej i góry
- 4 = Paeth - predyktor Paetha

### Przeplot Adam7

Metoda przeplotu dzieli obraz na 7 przebiegów:
```
Przebieg 1: co 8 piksel, początek (0,0)
Przebieg 2: co 8 piksel, początek (4,0)
Przebieg 3: co 4 piksel, początek (0,4)
Przebieg 4: co 4 piksel, początek (2,0)
Przebieg 5: co 2 piksel, początek (0,2)
Przebieg 6: co 2 piksel, początek (1,0)
Przebieg 7: co 1 piksel, początek (0,1)
```

To pozwala na progresywne ładowanie - najpierw gruby zarys, potem coraz więcej szczegółów.
