# C-code
It contains c codes I wrote at school. 

//             BIG CITY SIMULATION

#include <stdio.h>

#include <stdlib.h>

#include <string.h>

#include <time.h>

#include <ctype.h>


#ifdef _WIN32

    #include <windows.h>
    
    #include <conio.h>
    
    #define usleep(x) Sleep((x)/1000000) // usleep yerine Sleep kullanımı
    
#else

    #include <unistd.h>
    
    #include <sys/select.h>
    
#endif


#define WIDTH 57         // Haritanın genişliği (sütun sayısı)

#define HEIGHT 21        // Haritanın yüksekliği (satır sayısı)

#define MAX_PASSENGERS 8 // Bir otobüsün taşıyabileceği maksimum yolcu sayısı

#define MAX_LUGGAGE 8    // Bir otobüsün taşıyabileceği maksimum bagaj sayısı

#define MAX_TOTAL 200    // Simülasyonda olabilecek maksimum toplam yolcu sayısı

// ANSI Renk Kodları: Terminalde renkli çıktı vermek için kullanılır

#define RESET   "\033[0m"  // Rengi sıfırla

#define RED     "\033[31m" // Kırmızı renk

#define GREEN   "\033[32m" // Yeşil renk

#define YELLOW  "\033[33m" // Sarı renk

#define BLUE    "\033[34m" // Mavi renk

#define MAGENTA "\033[35m" // Macenta renk

#define CYAN    "\033[36m" // Turkuaz renk

#define WHITE   "\033[37m" // Beyaz renk

#define BOLD    "\033[1m"  // Kalın yazı

// Otobüs durağını temsil eden yapı (struct)

typedef struct {

    char name;               // Durağın adı (örn. 'A', 'B')
    int x, y;                // Haritadaki x ve y koordinatları
    int waitingPassengerIDs[20]; // Durakta bekleyen yolcuların ID'leri
    int count;               // Durakta bekleyen yolcu sayısı
} BusStop;

// Yolcuyu temsil eden yapı

typedef struct {

    int id;                  // Yolcunun benzersiz kimliği
    char line;               // Yolcunun binmek istediği otobüs hattı
    char startStop;          // Yolcunun başlangıç durağı
    char endStop;            // Yolcunun varış durağı
    int luggageCount;        // Yolcunun taşıdığı bagaj sayısı
    int luggageIDs[2];       // Yolcunun bagajlarının ID'leri (maks. 2 bagaj)
    int arrivalTime;         // Yolcunun simülasyona dahil olacağı zaman birimi
    int inBus;               // Yolcu otobüste mi (1: evet, 0: hayır)
    int placed;              // Yolcu durağa yerleştirildi mi (1: evet, 0: hayır - başlangıçta bekleme kuyruğunda)
} Passenger;

// Otobüsü temsil eden yapı

typedef struct {

    char id[3];              // Otobüsün kimliği (örn. "A1", "B2")
    char line;               // Otobüsün ait olduğu hat
    int direction;           // Otobüsün hareket yönü (1: ileri, -1: geri)
    int currentStopIndex;    // Otobüsün mevcut hattaki durak indeksini
    int posX, posY;          // Haritadaki mevcut x ve y koordinatları
    int waitTime;            // Otobüsün durakta bekleme süresi (birim)
    int passengerCount;      // Otobüsteki yolcu sayısı
    int passengerIDs[8];     // Otobüsteki yolcuların ID'leri
    int luggageCount;        // Otobüsteki bagaj sayısı
    int luggageIDs[8];       // Otobüsteki bagajların ID'leri
    int pathX[200], pathY[200]; // Gideceği yolun X ve Y dizileri
    int pathLen;                // Yolun uzunluğu
    int pathIndex;              // Şu an hangi adımda
    int moveWaitTime;
} Bus;

// Otobüs hattını temsil eden yapı

typedef struct {

    char name;               // Hattın adı (örn. 'A', 'B')
    char stops[10];          // Hattın üzerindeki duraklar dizisi
    int stopCount;           // Hattın üzerindeki durak sayısı
} BusLine;

// Seçim türünü belirten enum (numaralandırma)

typedef enum { NONE, BUS_SELECTED, STOP_SELECTED } SelectionType;

// Kullanıcının yaptığı seçimi (otobüs veya durak) temsil eden yapı
typedef struct {
    SelectionType type;      // Seçimin türü
    char name[3];            // Seçimin adı (otobüs ID'si veya durak adı)
} Selection;


// Fonksiyon prototipleri: Programdaki tüm fonksiyonların ön tanımları

void loadCityMap();            // Şehir haritasını yükler

void initBusStops();           // Otobüs duraklarını başlatır

void initBusLines();           // Otobüs hatlarını başlatır

void initBuses();              // Otobüsleri başlatır

void initPassengers();         // Yolcuları başlatır (ilk 15 yolcuyu oluşturur)

void simulateOneUnit();        // Simülasyonun bir zaman birimini ilerletir

void updateBuses();            // Otobüslerin konumlarını ve bekleme sürelerini günceller

void processPassengers();      // Yolcuları otobüslere bindirir/indirir

void generateNewPassengers();  // Yeni yolcuları oluşturur ve sisteme dahil eder

void printMap();               // Şehir haritasını ve üzerindeki otobüs/durak bilgilerini terminale yazdırır

void redrawScreen();           // Ekranı yeniden çizdirir (harita, istatistikler, detay bilgileri)

void printStats();             // Simülasyon istatistiklerini yazdırır

void displayBusInfo(char*);    // Seçilen otobüsün detaylı bilgilerini gösterir

void displayStopInfo(char);    // Seçilen durağın detaylı bilgilerini gösterir

void printPassengerQueue();    // Sisteme girmeyi bekleyen yeni yolcu kuyruğunu yazdırır

void handleTextCommand(char* input); // Kullanıcıdan gelen metin komutlarını işler

void clearInputBuffer();       // Giriş tamponunu temizler (Windows ve Unix uyumlu)


// Global değişkenler

char cityMap[HEIGHT][WIDTH + 1]; // Şehir haritasını tutan 2D dizi

BusLine lines[6];                // Tanımlanmış otobüs hatları dizisi

Bus buses[12];                   // Tanımlanmış otobüsler dizisi (her hattan 2 otobüs, 6 hat * 2 = 12)

Passenger allPassengers[MAX_TOTAL]; // Tüm yolcuları tutan dizi

BusStop busStops[16];            // Otobüs durakları dizisi

Selection currentSelection;      // Kullanıcının mevcut seçimini (otobüs/durak) tutar

int paused = 1;                  // Simülasyonun duraklatılıp duraklatılmadığı (1: duraklatıldı, 0: çalışıyor)

int passengerCount = 15;         // Mevcut toplam yolcu sayısı (allPassengers dizisinde kaç eleman dolu)

int luggageIDCounter = 100;      // Bagajlara benzersiz ID atamak için kullanılan sayaç


// Ana fonksiyon: Programın başlangıç noktası

// Ana fonksiyon: Programın başlangıç noktası

int main() {

    loadCityMap();
    initBusStops();
    initBusLines();
    initBuses();
    initPassengers();

    // Başlangıçta simülasyon duraklatılmış
    while (paused) {
        clearScreen();
        printBigCitySimulation();
        printf(BOLD GREEN "\nPress 'R' or 'r' to start the simulation..." RESET "\n");

        char ch = 0;
#ifdef _WIN32

        if (_kbhit()) ch = _getch();
#else

        struct timeval tv;
        fd_set fds;
        tv.tv_sec = 0;
        tv.tv_usec = 100000;
        FD_ZERO(&fds);
        FD_SET(STDIN_FILENO, &fds);
        select(STDIN_FILENO + 1, &fds, NULL, NULL, &tv);
        if (FD_ISSET(STDIN_FILENO, &fds)) ch = getchar();
#endif

        if (ch == 'r' || ch == 'R') {
            paused = 0;
        } else if (ch == 'q' || ch == 'Q') {
            printf(BOLD RED "\nSimulation ending..." RESET "\n");
            return 0;
        }
        usleep(50000);
    }

    printf(BOLD CYAN "=== BUS SIMULATION STARTED ===" RESET "\n");
    printf(GREEN "Commands:" RESET "\n");
    printf(YELLOW "R / r" RESET " - Toggle simulation (Run / Pause)\n");
    printf(YELLOW "Spacebar" RESET " - Advance one step (when paused)\n");
    printf(YELLOW "[A-P]" RESET " - Show detailed info for a bus stop\n");
    printf(YELLOW "[A-M][1-2]" RESET " - Show detailed info for a bus\n");
    printf(YELLOW "Q / q" RESET " - Quit simulation\n\n");

    redrawScreen();

#ifdef _WIN32

    static DWORD lastTick = 0;
#endif

    while (1) {
        int commandEntered = 0;
        char command[10] = {0};
        int needRedraw = 0;

#ifdef _WIN32

        if (_kbhit()) {
            char ch = _getch();
            if (ch == 'q' || ch == 'Q') {
                printf(BOLD RED "\nSimulation ending..." RESET "\n");
                return 0;
            } else if (ch == ' ') {
                strcpy(command, " ");
                commandEntered = 1;
            } else if (isalpha(ch)) {
                command[0] = toupper(ch);
                command[1] = '\0';
                if (_kbhit()) {
                    char ch2 = _getch();
                    if (ch2 == '1' || ch2 == '2') {
                        command[1] = ch2;
                        command[2] = '\0';
                    }
                }
                commandEntered = 1;
            } else if (ch == 'r' || ch == 'R') {
                strcpy(command, "r");
                commandEntered = 1;
            }
        }

        DWORD nowTick = GetTickCount();
        if (!paused && !commandEntered) {
            if (nowTick - lastTick >= 1000) { // 1 saniye geçtiyse
                simulateOneUnit();
                needRedraw = 1;
                lastTick = nowTick;
            }
        }
        if (!commandEntered && paused) {
            usleep(100000);
        }
#else

        fd_set fds;
        struct timeval tv;
        FD_ZERO(&fds);
        FD_SET(STDIN_FILENO, &fds);
        tv.tv_sec = 0;
        tv.tv_usec = (paused ? 0 : 100000);
        int retval = select(STDIN_FILENO + 1, &fds, NULL, NULL, &tv);
        if (retval == -1) {
            perror("select()");
            break;
        } else if (retval) {
            if (fgets(command, sizeof(command), stdin) != NULL) {
                command[strcspn(command, "\n")] = 0;
                if (strcmp(command, "q") == 0 || strcmp(command, "Q") == 0) {
                   printf(BOLD RED "\nSimulation ending..." RESET "\n");
                   return 0;
                }
                if (strlen(command) > 0 && isalpha(command[0])) {
                    command[0] = toupper(command[0]);
                }
                commandEntered = 1;
            }
        }
        if (!paused && !commandEntered) {
            simulateOneUnit();
            needRedraw = 1;
            usleep(1000000);
        }
#endif

        if (commandEntered) {
            if (strcmp(command, "r") == 0 || strcmp(command, "R") == 0) {
                paused = !paused;
                if (paused) {
                    printf(RED "[Simulation Paused]" RESET "\n");
                } else {
                    printf(GREEN "[Simulation Started]" RESET "\n");
                }
                needRedraw = 1;
            } else if (strcmp(command, " ") == 0) {
                if (paused) {
                    simulateOneUnit();
                    needRedraw = 1;
                } else {
                    paused = 1;
                    printf(RED "[Simulation Paused]" RESET "\n");
                    needRedraw = 1;
                }
            } else if (strlen(command) == 2 &&
                        isalpha(command[0]) &&
                        (command[1] == '1' || command[1] == '2')) {
                handleTextCommand(command);
                needRedraw = 1;
            } else if (strlen(command) == 1 && isalpha(command[0])) {
                handleTextCommand(command);
                needRedraw = 1;
            } else {
                printf(RED "Invalid command: %s" RESET "\n", command);
            }
        }

        if (needRedraw) {
            redrawScreen();
        }
    }
    return 0;
}

// ... (Diğer fonksiyonlar: clearScreen, loadCityMap, initBusStops vb. aynı kalacak) ...

void printBigCitySimulation() {

    printf(CYAN " ____  _____  _____ \n");
    printf(CYAN "|  _ \\|_   _|/ ____|" "\n");
    printf(CYAN "| |_) | | | | |  __ " "\n");
    printf(CYAN "|  _ <  | | | | |_ |" "\n");
    printf(CYAN "| |_) |_| |_| |__| |" "\n");
    printf(CYAN "|____/|_____|\\_____|" "\n");
    printf(RESET "\n");

    // CITY
    printf(CYAN "  _____ _____ _______   __" "\n");
    printf(CYAN " / ____|_   _|__   __|  \\ / /" "\n");
    printf(CYAN "| |      | |    | |      \\ / " "\n");
    printf(CYAN "| |      | |    | |      | |" "\n");
    printf(CYAN "| |____ _| |_   | |      | |" "\n");
    printf(CYAN " \\_____|_____|  |_|      |_|" "\n");
    printf(RESET "\n");

    // SIMULATION
    printf(CYAN " _____ _____ __  __ _    _ _            _______ _____ ____  _   _ " "\n");
    printf(CYAN "|  ___|_   _|  \\/  | |  | | |        /\\|__   __|_   _/ __ \\| \\ | |" "\n");
    printf(CYAN "| |___  | | | \\  / | |  | | |       /  \\  | |    | || |  | |  \\| |" "\n");
    printf(CYAN "|___  | | | | |\\/| | |  | | |      / /\\ \\ | |    | || |  | | . ` |" "\n");
    printf(CYAN " ___| |_| |_| |  | | |__| | |____ / ____ \\| |   _| || |__| | |\\  |" "\n");
    printf(CYAN "|_____|_____|_|  |_|\\____/|______/_/    \\_\\_|  |_____\\____/|_| \\_|" "\n");
}

void clearScreen() {

#ifdef _WIN32

    system("cls");
#else

    system("clear");
#endif

}

// Şehir haritasını yükleyen fonksiyon

void loadCityMap() {

    // Şehir haritasının sabit dize dizisi
    const char* map[HEIGHT] = {
        "#########################################################",
        "#A      #####B            #####C                 D#######",
        "# ########### ########### ##### ##### ########### #######",
        "# ########### ########### ##### ##### ########### #######",
        "# ########### ########### ##### ##### ########### #######",
        "# ########### ########### ##### ##### ########### #######",
        "# ########### ########### ##### ##### ########### #######",
        "# ########### ########### ##### ##### ########### #######",
        "#E           F                 G#####                 H      #",
        "# ########### ##### ########### ##### ##### ########### #",
        "# ########### ##### ########### ##### ##### ########### #",
        "# ########### ##### ########### ##### ##### ########### #",
        "# ########### ##### ########### ##### ##### ########### #",
        "# ########### ##### ########### ##### ##### ########### #",
        "#I           J                 K           L#####       #",
        "####### ##### ##### ########### ##### ##### ##### #######",
        "####### ##### ##### ########### ##### ##### ##### #######",
        "####### ##### ##### ########### ##### ##### ##### #######",
        "####### ##### ##### ########### ##### ##### ##### #######",
        "#      M     N      #####      O           P            #",
        "#########################################################"
    };
    // Diziyi global cityMap değişkenine kopyala
    for (int i = 0; i < HEIGHT; i++) {
        strncpy(cityMap[i], map[i], WIDTH + 1); // Her satırı kopyala
        cityMap[i][WIDTH] = '\0'; // Dizinin sonuna null sonlandırıcı ekle
    }
}

// Otobüs duraklarını başlatma ve konumlarını kaydetme fonksiyonu

void initBusStops() {

    int index = 0; // Durak dizisi indeksi
    // Harita üzerinde gezerek durak karakterlerini (A-P) bul
    for (int y = 0; y < HEIGHT; y++) {
        for (int x = 0; x < WIDTH; x++) {
            char ch = cityMap[y][x]; // Mevcut karakter
            if (ch >= 'A' && ch <= 'P') { // Eğer bir durak karakteriyse
                busStops[index].name = ch;     // Durağın adını kaydet
                busStops[index].x = x;         // x koordinatını kaydet
                busStops[index].y = y;         // y koordinatını kaydet
                busStops[index].count = 0;     // Başlangıçta durakta bekleyen yolcu sayısı 0
                index++;                       // Sonraki durak için indeksi artır
            }
        }
    }
}

// Otobüs hatlarını tanımlayan fonksiyon

void initBusLines() {

    // Hat A'nın tanımlanması
    lines[0].name = 'A';
    lines[0].stopCount = 6;
    char stopsA[] = {'A', 'E', 'I', 'J', 'K', 'L'}; // Durakları
    memcpy(lines[0].stops, stopsA, sizeof(stopsA)); // Durakları kopyala

    // Hat B'nin tanımlanması
    lines[1].name = 'B';
    lines[1].stopCount = 7;
    char stopsB[] = {'B', 'F', 'E', 'I', 'J', 'N', 'M'};
    memcpy(lines[1].stops, stopsB, sizeof(stopsB));

    // Hat C'nin tanımlanması
    lines[2].name = 'C';
    lines[2].stopCount = 7;
    char stopsC[] = {'C', 'G', 'K', 'J', 'F', 'E', 'A'};
    memcpy(lines[2].stops, stopsC, sizeof(stopsC));

    // Hat D'nin tanımlanması
    lines[3].name = 'D';
    lines[3].stopCount = 7;
    char stopsD[] = {'D', 'C', 'G', 'K', 'J', 'N', 'M'};
    memcpy(lines[3].stops, stopsD, sizeof(stopsD));

    // Hat L'nin tanımlanması
    lines[4].name = 'L';
    lines[4].stopCount = 8;
    char stopsL[] = {'L', 'P', 'O', 'K', 'G', 'C', 'D', 'H'};
    memcpy(lines[4].stops, stopsL, sizeof(stopsL));

    // Hat M'nin tanımlanması
    lines[5].name = 'M';
    lines[5].stopCount = 8;
    char stopsM[] = {'M', 'N', 'J', 'F', 'G', 'K', 'O', 'P'};
    memcpy(lines[5].stops, stopsM, sizeof(stopsM));
}

// Otobüsleri başlatan ve başlangıç konumlarını ayarlayan fonksiyon

void initBuses() {

    int busIndex = 0; // Otobüs dizisi indeksi
    for (int i = 0; i < 6; i++) { // Her hat için
        BusLine line = lines[i]; // Mevcut hattı al

        // Birinci otobüs (ileriye gider)
        Bus* bus1 = &buses[busIndex++]; // Otobüse referans al ve indeksi artır
        sprintf(bus1->id, "%c1", line.name); // ID'yi ayarla (örn. "A1")
        bus1->line = line.name;             // Hatta ata
        bus1->direction = 1;                // İleri yöne ayarla
        bus1->currentStopIndex = 0;         // İlk durak indeksini ayarla

        // Başlangıç durağının koordinatlarını bul
        char startStop = line.stops[0]; // Hattın ilk durağı
        for (int s = 0; s < 16; s++) {  // Tüm duraklar arasında gez
            if (busStops[s].name == startStop) { // Eğer eşleşen durak bulunursa
                bus1->posX = busStops[s].x; // Otobüsün x koordinatını ayarla
                bus1->posY = busStops[s].y; // Otobüsün y koordinatını ayarla
                break;                      // Döngüden çık
            }
        }
        bus1->waitTime = 0;         // Başlangıçta bekleme süresi yok
        bus1->passengerCount = 0;   // Başlangıçta yolcu yok
        bus1->luggageCount = 0;     // Başlangıçta bagaj yok

        // İkinci otobüs (geriye gider)
        Bus* bus2 = &buses[busIndex++]; // Otobüse referans al ve indeksi artır
        sprintf(bus2->id, "%c2", line.name); // ID'yi ayarla (örn. "A2")
        bus2->line = line.name;             // Hatta ata
        bus2->direction = -1;               // Geri yöne ayarla
        bus2->currentStopIndex = line.stopCount - 1; // Son durak indeksini ayarla
        // Son duraktaki koordinatları bul
        char endStop = line.stops[line.stopCount - 1]; // Hattın son durağı
        for (int s = 0; s < 16; s++) {  // Tüm duraklar arasında gez
            if (busStops[s].name == endStop) { // Eğer eşleşen durak bulunursa
                bus2->posX = busStops[s].x; // Otobüsün x koordinatını ayarla
                bus2->posY = busStops[s].y; // Otobüsün y koordinatını ayarla
                break;                      // Döngüden çık
            }
        }
        bus2->waitTime = 0;         // Başlangıçta bekleme süresi yok
        bus2->passengerCount = 0;   // Başlangıçta yolcu yok
        bus2->luggageCount = 0;     // Başlangıçta bagaj yok
        bus2->moveWaitTime = 0;     // Hareket bekleme süresi sıfır

    }
}

// İlk yolcuları oluşturan ve başlangıç değerlerini ayarlayan fonksiyon

void initPassengers() {

    srand(time(NULL)); // Rastgele sayı üretecinin tohumunu ayarla (her çalıştırmada farklı sonuçlar için)
    for (int i = 0; i < 15; i++) { // İlk 15 yolcuyu oluştur
        Passenger* p = &allPassengers[i]; // Yolcuya referans al
        p->id = i;                         // Yolcu ID'sini ayarla

        // Rastgele bir hat seç
        int lineIndex = rand() % 6; // 0-5 arası rastgele bir hat indeksi
        BusLine line = lines[lineIndex]; // Seçilen hat
        p->line = line.name;             // Yolcunun binmek istediği hattı ata

        // Rastgele farklı 2 durak seç (başlangıç ve varış)
        int from, to;
        do {
            from = rand() % line.stopCount; // Başlangıç durağı indeksi
            to = rand() % line.stopCount;   // Varış durağı indeksi
        } while (from == to); // Başlangıç ve varış durakları farklı olmalı

        p->startStop = line.stops[from]; // Başlangıç durağını ata
        p->endStop = line.stops[to];     // Varış durağını ata

        // Valiz sayısı: 0-2 arası rastgele
        p->luggageCount = rand() % 3; // 0, 1 veya 2
        for (int j = 0; j < p->luggageCount; j++) { // Her valiz için
            p->luggageIDs[j] = luggageIDCounter++; // Benzersiz bagaj ID'si ata ve sayacı artır
        }

        // Simülasyona ne zaman gireceği: 0-3 birim sonra
        p->arrivalTime = rand() % 4; // Mevcut zamana eklenecek bekleme süresi
        p->inBus = 0;                // Başlangıçta otobüste değil
        p->placed = 0;               // Başlangıçta durağa yerleştirilmedi
    }
    passengerCount = 15; // Başlangıçta toplam yolcu sayısı 15
}

// Simülasyonun bir zaman birimini ilerleten ana fonksiyon

void simulateOneUnit() {

    currentTime++;           // Zamanı bir birim artır
    generateNewPassengers(); // Yeni yolcuları oluştur ve sisteme dahil et
    updateBuses();           // Otobüslerin konumlarını güncelle
    processPassengers();     // Yolcuları indir/bindir
}

// Yeni yolcuları oluşturan ve duraklara yerleştiren fonksiyon

void generateNewPassengers() {

    // Mevcut yolcuları kontrol et: Zamanı gelmiş ve henüz durağa yerleştirilmemiş yolcuları duraklara yerleştir
    for (int i = 0; i < passengerCount; i++) { // Mevcut tüm yolcuları gez
        Passenger* p = &allPassengers[i];      // Yolcuya referans al
        // Yolcunun simülasyona dahil olma zamanı gelmişse VE durağa yerleştirilmemişse VE otobüste değilse
        if (p->arrivalTime <= currentTime && !p->placed && !p->inBus) {
            // Bu yolcu artık simülasyona girebilir -> başlangıç durağının listesine ekle
            for (int j = 0; j < 16; j++) { // Tüm durakları gez
                if (busStops[j].name == p->startStop) { // Eğer yolcunun başlangıç durağı bulunursa
                    int currentStopCount = busStops[j].count; // Durağın mevcut yolcu sayısı
                    if (currentStopCount < 20) { // Durağın kapasitesi dolmadıysa
                        busStops[j].waitingPassengerIDs[currentStopCount] = p->id; // Yolcuyu durağa ekle
                        busStops[j].count++; // Durağın yolcu sayısını artır
                        p->placed = 1;       // Yolcuyu durağa yerleştirildi olarak işaretle
                        break;               // Eklendiği için döngüden çık
                    }
                }
            }
        }
    }

    // Yeni yolcu oluştur (sınır kontrolü ile ve olasılığa bağlı olarak)
    if (passengerCount < MAX_TOTAL) { // Toplam yolcu sayısı üst sınırı aşmadıysa
        if (rand() % 10 == 0) { // Her 10 zaman biriminden birinde yeni yolcu gelme olasılığı (ayarlanabilir)
            Passenger* newP = &allPassengers[passengerCount]; // Yeni yolcu için yer aç
            newP->id = passengerCount; // Yeni yolcuya ID ata

            int lineIndex = rand() % 6;       // Rastgele bir hat seç
            BusLine line = lines[lineIndex];  // Seçilen hat
            newP->line = line.name;           // Yolcunun hattını ata

            int from, to;
            do {
                from = rand() % line.stopCount; // Başlangıç durağı indeksi
                to = rand() % line.stopCount;   // Varış durağı indeksi
            } while (from == to); // Başlangıç ve varış durakları farklı olmalı

            newP->startStop = line.stops[from]; // Başlangıç durağını ata
            newP->endStop = line.stops[to];     // Varış durağını ata
            newP->luggageCount = rand() % 3;    // Valiz sayısı (0-2)
            for (int l = 0; l < newP->luggageCount; l++) { // Her valiz için
                newP->luggageIDs[l] = luggageIDCounter++; // Benzersiz bagaj ID'si ata
            }

            // Simülasyona dahil olma zamanı: Mevcut zamandan 1-5 birim sonra
            newP->arrivalTime = currentTime + (rand() % 5) + 1;
            newP->inBus = 0;  // Başlangıçta otobüste değil
            newP->placed = 0; // Başlangıçta durağa yerleştirilmedi

            passengerCount++; // Toplam yolcu sayısını artır
        }
    }
}


// Yolcuları otobüslere bindiren ve indiren fonksiyon

void processPassengers() {

    for (int b = 0; b < 12; b++) { // Tüm otobüsleri gez
        Bus* bus = &buses[b];      // Otobüse referans al

        // Otobüsün ait olduğu hattı bul
        BusLine* line = NULL;
        for (int l = 0; l < 6; l++) { // Tüm hatları gez
            if (lines[l].name == bus->line) { // Hat bulunduysa
                line = &lines[l];             // Hatta referans al
                break;                        // Döngüden çık
            }
        }
        if (!line) continue; // Hat bulunamadıysa, sonraki otobüse geç

        char currentStop = line->stops[bus->currentStopIndex]; // Otobüsün mevcut durağı

        // === 1. Yolcuları indir ===
        // Otobüs içindeki yolcuları gez
        for (int p_idx_in_bus = 0; p_idx_in_bus < bus->passengerCount;) {
            int pid = bus->passengerIDs[p_idx_in_bus]; // Otobüsteki yolcunun ID'si

            // PID'nin geçerli olup olmadığını kontrol et (güvenlik için)
            if (pid >= passengerCount || pid < 0) {
                // Geçersiz yolcu ID'si ise, otobüs listesinden sil
                for (int k = p_idx_in_bus; k < bus->passengerCount - 1; k++) {
                    bus->passengerIDs[k] = bus->passengerIDs[k + 1];
                }
                bus->passengerCount--; // Yolcu sayısını azalt
                continue; // Sonraki yolcuya geç (indeks aynı kalır)
            }

            Passenger* passenger = &allPassengers[pid]; // Yolcu nesnesine referans al

            // Eğer yolcunun varış durağı mevcut duraksa VE yolcu otobüste ise
            if (passenger->endStop == currentStop && passenger->inBus) {
                // Yolcunun valizlerini indir (otobüsün bagaj listesinden sil)
                for (int l = 0; l < passenger->luggageCount; l++) {
                    int lid = passenger->luggageIDs[l]; // Yolcunun bagaj ID'si
                    for (int i = 0; i < bus->luggageCount; i++) { // Otobüsün tüm bagajlarını gez
                        if (bus->luggageIDs[i] == lid) { // Bagaj bulunduysa
                            // Bagajı otobüs listesinden sil
                            for (int k = i; k < bus->luggageCount - 1; k++) {
                                bus->luggageIDs[k] = bus->luggageIDs[k + 1];
                            }
                            bus->luggageCount--; // Otobüsün bagaj sayısını azalt
                            break; // Bu bagaj silindi, diğerine geç
                        }
                    }
                }

                // Yolcuyu otobüsten indir (otobüsün yolcu listesinden sil)
                for (int k = p_idx_in_bus; k < bus->passengerCount - 1; k++) {
                    bus->passengerIDs[k] = bus->passengerIDs[k + 1];
                }
                bus->passengerCount--; // Otobüsün yolcu sayısını azalt
                passenger->inBus = 0;  // Yolcuyu otobüste değil olarak işaretle
                passenger->placed = 0; // Yolcuyu "yerleştirilmedi" olarak işaretle (sistemden çıktı)
            } else {
                p_idx_in_bus++; // Yolcu indirilmediyse, sonraki yolcuya geç
            }
        }

        // === 2. Şu anki durağı bul ===
        BusStop* stop = NULL;
        for (int s = 0; s < 16; s++) { // Tüm durakları gez
            if (busStops[s].name == currentStop) { // Eğer mevcut durak bulunursa
                stop = &busStops[s];               // Durağa referans al
                break;                             // Döngüden çık
            }
        }
        if (!stop) continue; // Durak bulunamadıysa (olmamalı), sonraki otobüse geç

        // === 3. Yolcuları bindir ===
        // Durakta bekleyen yolcuları gez
        for (int s_idx_in_stop = 0; s_idx_in_stop < stop->count;) {
            // Otobüs kapasitesi veya bagaj kapasitesi dolduysa, bindirmeyi durdur
            if (bus->passengerCount >= MAX_PASSENGERS || bus->luggageCount >= MAX_LUGGAGE) break;

            int pid = stop->waitingPassengerIDs[s_idx_in_stop]; // Duraktaki yolcunun ID'si

            // PID'nin geçerli olup olmadığını kontrol et (güvenlik için)
            if (pid >= passengerCount || pid < 0) {
                // Geçersiz yolcu ID'si ise, durağın listesinden sil
                for (int k = s_idx_in_stop; k < stop->count - 1; k++) {
                    stop->waitingPassengerIDs[k] = stop->waitingPassengerIDs[k + 1];
                }
                stop->count--; // Durağın yolcu sayısını azalt
                continue;      // Sonraki yolcuya geç (indeks aynı kalır)
            }

            Passenger* passenger = &allPassengers[pid]; // Yolcu nesnesine referans al

            // Yolcu zaten otobüste mi? (Bu durum olmamalı ama güvenlik için)
            if (passenger->inBus) {
                // Duraktaki listeden sil
                for (int k = s_idx_in_stop; k < stop->count - 1; k++) {
                    stop->waitingPassengerIDs[k] = stop->waitingPassengerIDs[k + 1];
                }
                stop->count--; // Durağın yolcu sayısını azalt
                continue;      // Sonraki yolcuya geç (indeks aynı kalır)
            }

            // Yolcunun gitmek istediği hat otobüsün hattıyla uyuyor mu?
            if (passenger->line != bus->line) {
                s_idx_in_stop++; // Uymuyorsa, sonraki yolcuya geç
                continue;
            }

            // Yolcunun varış durağı bu otobüs hattında var mı?
            int validStop = 0;
            for (int j = 0; j < line->stopCount; j++) { // Hattın tüm duraklarını gez
                if (line->stops[j] == passenger->endStop) { // Varış durağı bulunursa
                    validStop = 1; // Geçerli olarak işaretle
                    break;         // Döngüden çık
                }
            }
            if (!validStop) { // Geçerli varış durağı bulunamadıysa
                s_idx_in_stop++; // Sonraki yolcuya geç
                continue;
            }

            // Yolcunun valizleri otobüse sığar mı?
            if (bus->luggageCount + passenger->luggageCount > MAX_LUGGAGE) {
                s_idx_in_stop++; // Sığmıyorsa, sonraki yolcuya geç
                continue;
            }

            // Yolcuyu otobüse bindir
            bus->passengerIDs[bus->passengerCount++] = pid; // Yolcuyu otobüsün listesine ekle ve yolcu sayısını artır
            for (int j = 0; j < passenger->luggageCount; j++) { // Yolcunun bagajlarını otobüse ekle
                bus->luggageIDs[bus->luggageCount++] = passenger->luggageIDs[j]; // Bagajı otobüsün listesine ekle
            }
            passenger->inBus = 1;  // Yolcuyu otobüste olarak işaretle
            passenger->placed = 1; // Yolcuyu hala "yerleştirildi" olarak işaretle (çünkü otobüste)

            // Yolcuyu durağın bekleme listesinden sil
            for (int k = s_idx_in_stop; k < stop->count - 1; k++) {
                stop->waitingPassengerIDs[k] = stop->waitingPassengerIDs[k + 1];
            }
            stop->count--; // Durağın yolcu sayısını azalt
            // s_idx_in_stop aynı kalır, çünkü silme işleminden sonra bir sonraki öğe mevcut indekse kaydırıldı
        }
    }
}

// Şehir haritasını ve üzerindeki öğeleri terminale yazdıran fonksiyon

void printMap() {



    // Haritayı her çizim için orijinal halinden kopyala
    char tempCityMap[HEIGHT][WIDTH + 1]; // Geçici bir harita kopyası oluştur
    for (int i = 0; i < HEIGHT; i++) {
        strncpy(tempCityMap[i], cityMap[i], WIDTH + 1); // Orijinal haritayı temp'e kopyala
    }

    // 1. Duraklara bekleyen yolcu sayılarını yaz
    for (int i = 0; i < 16; i++) { // Tüm durakları gez
        BusStop* stop = &busStops[i]; // Durağa referans al
        int px = stop->x;             // Durağın x koordinatı
        int py = stop->y;             // Durağın y koordinatı

        // Bekleyen yolcu sayısını karakter olarak al (0-9 arası sayı, fazlası için '*')
        char countChar = (stop->count >= 0 && stop->count <= 9) ?
                         (stop->count + '0') : '*';

        // Durağın sağındaki '#' karaktere yazmaya çalış
        if (px + 1 < WIDTH && tempCityMap[py][px + 1] == '#') {
            tempCityMap[py][px + 1] = countChar;
        }
        // Durağın altındaki '#' karaktere yazmaya çalış
        else if (py + 1 < HEIGHT && tempCityMap[py + 1][px] == '#') {
            tempCityMap[py + 1][px] = countChar;
        }
        // Alternatif olarak, durağın sağındaki veya solundaki boş bir alana yazmaya çalış
        else if (px + 2 < WIDTH && tempCityMap[py][px + 2] == ' ') { // Sağa iki boşluk
            tempCityMap[py][px + 2] = countChar;
        }
        else if (px - 2 >= 0 && tempCityMap[py][px - 2] == ' ') { // Sola iki boşluk
            tempCityMap[py][px - 2] = countChar;
        }
        // Diğer durumlar (örneğin uygun boşluk bulunamadıysa) bu sayıyı göstermez
    }

    // 2. Otobüsleri haritaya yaz
    for (int i = 0; i < 12; i++) { // Tüm otobüsleri gez
        Bus* bus = &buses[i];      // Otobüse referans al
        int px = bus->posX;        // Otobüsün x koordinatı
        int py = bus->posY;        // Otobüsün y koordinatı

        // Koordinatların harita sınırları içinde olduğunu kontrol et
        if (px >= 0 && px < WIDTH && py >= 0 && py < HEIGHT) {
            // Otobüsün yolcu sayısını sembol olarak al (0-9 arası sayı, 0 için 'o', fazlası için '*')
            char symbol = (bus->passengerCount >= 0 && bus->passengerCount <= 9)
                          ? (bus->passengerCount + '0') : '*';
            if (bus->passengerCount == 0) symbol = 'o'; // Yolcu yoksa 'o' sembolü

            // Otobüsün bir durağın üzerinde olup olmadığını kontrol et
            int onStop = 0;
            for (int s = 0; s < 16; s++) { // Tüm durakları gez
                if (busStops[s].x == px && busStops[s].y == py) { // Otobüs durağın üzerindeyse
                    onStop = 1; // İşaretle
                    break;      // Döngüden çık
                }
            }

            if (onStop) {
                // Eğer otobüs durağın üzerindeyse, durak harfini koru
                // Ve otobüs sembolünü durak harfinin hemen yanına yerleştirmeye çalış
                if (px + 1 < WIDTH && tempCityMap[py][px + 1] != '#' && !isdigit(tempCityMap[py][px+1]) && tempCityMap[py][px+1] != 'o') {
                    tempCityMap[py][px + 1] = symbol;
                } else if (px - 1 >= 0 && tempCityMap[py][px - 1] != '#' && !isdigit(tempCityMap[py][px-1]) && tempCityMap[py][px-1] != 'o') {
                    tempCityMap[py][px - 1] = symbol;
                } else if (py + 1 < HEIGHT && tempCityMap[py + 1][px] != '#' && !isdigit(tempCityMap[py+1][px]) && tempCityMap[py+1][px] != 'o') {
                    tempCityMap[py + 1][px] = symbol;
                } else if (py - 1 >= 0 && tempCityMap[py - 1][px] != '#' && !isdigit(tempCityMap[py-1][px]) && tempCityMap[py-1][px] != 'o') {
                    tempCityMap[py - 1][px] = symbol;
                }
                // Eğer durak üzerinde ve etrafına da koyamıyorsak, otobüs bu karede görünmez kalır.
                // Durak karakterinin üzerine yazılmaz.
            } else {
                // Otobüs durağın üzerinde değilse, doğrudan kendi konumuna yerleştir
                tempCityMap[py][px] = symbol;
            }
        }
    }

    // 3. Haritayı renkli yazdır
    for (int i = 0; i < HEIGHT; i++) { // Her satırı gez
        for (int j = 0; j < WIDTH; j++) { // Her sütunu gez
            char ch = tempCityMap[i][j]; // Geçici haritadan karakteri al

            if (ch >= 'A' && ch <= 'P') { // Eğer karakter bir durak harfiyse
                printf(BOLD GREEN "%c" RESET, ch); // Kalın yeşil yazdır
            } else if (ch >= '0' && ch <= '9') { // Eğer karakter bir sayıysa
                // Sayının bir durak yolcu sayısı mı yoksa otobüs yolcu sayısı mı olduğunu kontrol et
                int isStopCount = 0;
                for(int s=0; s<16; s++){ // Tüm durakları gez
                    // Eğer sayı durağın sağında, altında, iki sağında veya iki solunda ise durak sayısıdır
                    if((busStops[s].x + 1 == j && busStops[s].y == i) ||
                       (busStops[s].y + 1 == i && busStops[s].x == j) ||
                       (busStops[s].x + 2 == j && busStops[s].y == i) ||
                       (busStops[s].x - 2 == j && busStops[s].y == i) ){
                        isStopCount = 1; // Durak sayısı olarak işaretle
                        break;           // Döngüden çık
                    }
                }
                if(isStopCount){
                     printf(BOLD YELLOW "%c" RESET, ch); // Durak sayısıysa kalın sarı yazdır
                } else { // Değilse, otobüs yolcu sayısıdır
                     printf(BOLD RED "%c" RESET, ch);   // Kalın kırmızı yazdır
                }
            } else if (ch == 'o') { // Eğer karakter 'o' ise (yolcusuz otobüs)
                printf(BOLD RED "%c" RESET, ch); // Kalın kırmızı yazdır
            } else if (ch == '#') { // Eğer karakter '#' ise (duvar)
                printf(BLUE "%c" RESET, ch);     // Mavi yazdır
            } else { // Diğer karakterler (boşluk vb.)
                printf("%c", ch);                // Olduğu gibi yazdır
            }
        }
        printf("\n"); // Her satırın sonunda yeni satıra geç
    }
}

// Ekranı yeniden çizen ve tüm bilgileri güncelleyen fonksiyon

void redrawScreen() {

    clearScreen(); // Ekranı temizle
    printMap();      // Haritayı çiz
    printStats();    // İstatistikleri yazdır
    // Eğer otobüs seçiliyse, otobüs bilgisini göster
    if (currentSelection.type == BUS_SELECTED)
        displayBusInfo(currentSelection.name);
    // Eğer durak seçiliyse, durak bilgisini göster
    else if (currentSelection.type == STOP_SELECTED)
        displayStopInfo(currentSelection.name[0]);
    printPassengerQueue(); // Yeni yolcu kuyruğunu yazdır
    printf(CYAN ">> " RESET); // Komut istemini tekrar göster
    fflush(stdout); // Çıkış tamponunu temizle, böylece çıktılar hemen görünür
}

// Simülasyon istatistiklerini yazdıran fonksiyon

void printStats() {

    int waitingPassengers = 0;       // Duraklarda bekleyen yolcu sayısı
    int onboardPassengers = 0;       // Otobüslerdeki yolcu sayısı
    int totalLuggage = 0;            // Toplam bagaj sayısı
    int totalNewPassengersInQueue = 0; // Sisteme girmeyi bekleyen yeni yolcu sayısı

    // Durağa ulaşmış ama otobüse binmemiş yolcuları ve bagajlarını say
    for (int i = 0; i < 16; i++) { // Tüm durakları gez
        waitingPassengers += busStops[i].count; // Duraktaki yolcu sayısını ekle
        for (int j = 0; j < busStops[i].count; j++) { // Duraktaki her yolcunun bagajını say
            int pid = busStops[i].waitingPassengerIDs[j]; // Yolcu ID'si
            if (pid >= 0 && pid < passengerCount) { // Geçerli ID kontrolü
                totalLuggage += allPassengers[pid].luggageCount; // Yolcunun bagaj sayısını topla
            }
        }
    }

    // Otobüs içindeki yolcuları ve bagajlarını say
    for (int i = 0; i < 12; i++) { // Tüm otobüsleri gez
        onboardPassengers += buses[i].passengerCount; // Otobüsteki yolcu sayısını ekle
        totalLuggage += buses[i].luggageCount;       // Otobüsteki bagaj sayısını topla
    }

    // Henüz durağa yerleştirilmemiş ve otobüste olmayan yeni yolcuları say
    for (int i = 0; i < passengerCount; i++) { // Mevcut tüm yolcuları gez
        Passenger* p = &allPassengers[i];      // Yolcuya referans al
        if (!p->placed && !p->inBus) { // Eğer yolcu durağa yerleştirilmemişse VE otobüste değilse
            totalNewPassengersInQueue++; // Kuyruktaki yeni yolcu sayısını artır
        }
    }

    // İstatistikleri terminale yazdır
    printf(BOLD CYAN "\n=== Simulation Time: %d ===" RESET "\n", currentTime); // Mevcut zamanı göster
    printf(GREEN "New Passengers in Queue : %d" RESET "\n", totalNewPassengersInQueue); // Yeni yolcu kuyruğu sayısı
    printf(YELLOW "Waiting at Stops    : %d" RESET "\n", waitingPassengers); // Duraklarda bekleyen sayısı
    printf(MAGENTA "Onboard Passengers  : %d" RESET "\n", onboardPassengers); // Otobüslerdeki yolcu sayısı
    printf(BLUE "Total Luggage       : %d" RESET "\n", totalLuggage);     // Toplam bagaj sayısı
    printf(CYAN "===========================" RESET "\n"); // Ayırıcı
}

// Seçilen otobüsün detaylı bilgilerini gösteren fonksiyon

void displayBusInfo(char* id) {

    Bus* bus = NULL;             // Otobüs pointer'ı
    for (int i = 0; i < 12; i++) { // Tüm otobüsleri gez
        if (strcmp(buses[i].id, id) == 0) { // Eğer ID eşleşirse
            bus = &buses[i];             // Otobüsü bul
            break;                       // Döngüden çık
        }
    }

    if (!bus) { // Otobüs bulunamadıysa
        printf(RED "No such bus found: %s" RESET "\n", id); // Hata mesajı
        return;                                         // Fonksiyondan çık
    }

    BusLine* line = NULL;      // Hat pointer'ı
    for (int l = 0; l < 6; l++) { // Tüm hatları gez
        if (lines[l].name == bus->line) { // Otobüsün hattı bulunursa
            line = &lines[l];             // Hatta referans al
            break;                        // Döngüden çık
        }
    }

    char currentStop = '?'; // Mevcut durak karakteri (varsayılan '?')
    if (line) { // Hat bulunursa
        currentStop = line->stops[bus->currentStopIndex]; // Mevcut durağın karakterini al
    }

    // Otobüs bilgilerini yazdır
    printf(BOLD YELLOW "\n--- Bus Info [%s] ---" RESET "\n", bus->id); // Otobüs ID'si
    printf(GREEN "Line       : %c" RESET "\n", bus->line);             // Hattı
    printf(BLUE "Direction  : %s" RESET "\n", bus->direction == 1 ? "Forward" : "Reverse"); // Yönü
    printf(CYAN "Current Stop: %c" RESET "\n", currentStop);         // Mevcut durağı

    printf(MAGENTA "Passengers : " RESET); // Yolcu başlığı
    if (bus->passengerCount == 0) { // Yolcu yoksa
        printf("None");             // "None" yazdır
    } else {                        // Yolcu varsa
        for (int i = 0; i < bus->passengerCount; i++) { // Her yolcunun ID'sini yazdır
            printf("%d ", bus->passengerIDs[i]);
        }
    }
    printf("\n"); // Yeni satır

    printf(YELLOW "Luggage    : " RESET); // Bagaj başlığı
    if (bus->luggageCount == 0) { // Bagaj yoksa
        printf("None");           // "None" yazdır
    } else {                      // Bagaj varsa
        for (int i = 0; i < bus->luggageCount; i++) { // Her bagajın ID'sini yazdır
            printf("%d ", bus->luggageIDs[i]);
        }
    }
    printf("\n" YELLOW "------------------------" RESET "\n"); // Ayırıcı
}

// Seçilen durağın detaylı bilgilerini gösteren fonksiyon

void displayStopInfo(char stopName) {

    BusStop* stop = NULL;            // Durak pointer'ı
    for (int i = 0; i < 16; i++) {   // Tüm durakları gez
        if (busStops[i].name == stopName) { // Eğer durak adı eşleşirse
            stop = &busStops[i];          // Durağı bul
            break;                        // Döngüden çık
        }
    }

    if (!stop) { // Durak bulunamadıysa
        printf(RED "No such stop: %c" RESET "\n", stopName); // Hata mesajı
        return;                                         // Fonksiyondan çık
    }

    // Durak bilgilerini yazdır
    printf(BOLD CYAN "\n--- Stop Info [%c] ---" RESET "\n", stop->name); // Durak adı
    printf(GREEN "Waiting Passengers: " RESET); // Bekleyen yolcu başlığı
    if (stop->count == 0) { // Bekleyen yolcu yoksa
        printf("None");     // "None" yazdır
    } else {                // Bekleyen yolcu varsa
        for (int i = 0; i < stop->count; i++) { // Her yolcunun bilgilerini yazdır
            int pid = stop->waitingPassengerIDs[i]; // Yolcu ID'si
            if (pid >= 0 && pid < passengerCount) { // Geçerli ID kontrolü
                Passenger* p = &allPassengers[pid]; // Yolcuya referans al
                // Yolcu ID, hattı, başlangıç-varış durakları ve bagaj sayısı
                printf("%d (Line:%c %c->%c L:%d) ", p->id, p->line, p->startStop, p->endStop, p->luggageCount);
            }
        }
    }
    printf("\n" CYAN "------------------------" RESET "\n"); // Ayırıcı
}

// Sisteme girmeyi bekleyen yeni yolcu kuyruğunu yazdıran fonksiyon

void printPassengerQueue() {

    printf(BOLD MAGENTA "\n--- New Passengers Queue ---" RESET "\n"); // Kuyruk başlığı
    int count = 0; // Yazdırılan yolcu sayısı sayacı
    for (int i = 0; i < passengerCount; i++) { // Mevcut tüm yolcuları gez
        Passenger* p = &allPassengers[i];      // Yolcuya referans al
        // Eğer yolcu durağa yerleştirilmemişse VE otobüste değilse VE henüz varış zamanı gelmemişse
        if (!p->placed && !p->inBus && p->arrivalTime > currentTime) {
            // Yolcu ID, hattı, başlangıç-varış durakları, bagaj sayısı ve varış zamanı
            printf(YELLOW "ID:%d (Line:%c %c->%c L:%d Arr:%d)" RESET "\n",
                   p->id, p->line, p->startStop, p->endStop, p->luggageCount, p->arrivalTime);
            count++; // Sayacı artır
            if (count >= 5) break; // İlk 5 yeni yolcuyu göster (limit)
        }
    }
    if (count == 0) { // Kuyrukta yolcu yoksa
        printf("No new passengers in queue." "\n"); // Mesaj yazdır
    }
    printf(MAGENTA "----------------------------" RESET "\n"); // Ayırıcı
}

// Kullanıcıdan gelen metin komutlarını işleyen fonksiyon

void handleTextCommand(char* input) {

    if (strlen(input) == 1 && isalpha(input[0])) { // Tek karakterli ve harf mi?
        // Durak seçimi
        currentSelection.type = STOP_SELECTED;     // Seçim türünü durağa ayarla
        currentSelection.name[0] = toupper(input[0]); // Harfi büyüt ve adı kaydet
        currentSelection.name[1] = '\0';           // Dize sonlandırıcı ekle
    } else if (strlen(input) == 2 && isalpha(input[0]) && (input[1] == '1' || input[1] == '2')) { // İki karakterli mi ve belirli formata mı uyuyor?
        // Otobüs seçimi
        currentSelection.type = BUS_SELECTED;      // Seçim türünü otobüse ayarla
        currentSelection.name[0] = toupper(input[0]); // İlk harfi büyüt
        currentSelection.name[1] = input[1];       // İkinci karakteri kaydet
        currentSelection.name[2] = '\0';           // Dize sonlandırıcı ekle
    } else {
        currentSelection.type = NONE;              // Geçersiz komut, seçimi sıfırla
    }
}

// Giriş tamponunu temizleyen yardımcı fonksiyon

void clearInputBuffer() {

    int c;
    while ((c = getchar()) != '\n' && c != EOF); // Yeni satır veya dosya sonu karakterine kadar oku ve at
}

// İki nokta arasında düz yol (sağa/sola ve yukarı/aşağı) oluşturan basit yol bulucu

int findPath(int x1, int y1, int x2, int y2, int* pathX, int* pathY) {

    int len = 0;
    int cx = x1, cy = y1;
    // X ekseninde ilerle
    while (cx != x2) {
        cx += (x2 > cx) ? 1 : -1;
        pathX[len] = cx;
        pathY[len] = cy;
        len++;
    }
    // Y ekseninde ilerle
    while (cy != y2) {
        cy += (y2 > cy) ? 1 : -1;
        pathX[len] = cx;
        pathY[len] = cy;
        len++;
    }
    return len;
}

// Otobüslerin hareketini ve bekleme sürelerini güncelleyen fonksiyon (adım adım ilerleme ile)

void updateBuses() {

    for (int i = 0; i < 12; i++) {
        Bus* bus = &buses[i];

        // 1. Eğer otobüs durakta bekliyorsa, bekleme süresini azalt
        if (bus->waitTime > 0) {
            bus->waitTime--;
            continue;
        }

        // 2. Hareket arası bekleme (yolda adım adım ilerleme)
        if (bus->moveWaitTime > 0) {
            bus->moveWaitTime--;
            continue;
        }

        // 3. Eğer otobüs bir yol üzerinde ilerliyorsa
        if (bus->pathIndex < bus->pathLen) {
            bus->posX = bus->pathX[bus->pathIndex];
            bus->posY = bus->pathY[bus->pathIndex];
            bus->pathIndex++;
            bus->moveWaitTime = 1; // Her adımda bir kare ilerle
            // Eğer yolun sonuna geldiyse, durakta bekleme başlat
            if (bus->pathIndex == bus->pathLen) {
                bus->waitTime = 3; // Durağa varınca 3 birim bekle
            }
            continue;
        }

        // 4. Otobüsün ait olduğu hattı bul
        BusLine* line = NULL;
        for (int l = 0; l < 6; l++) {
            if (lines[l].name == bus->line) {
                line = &lines[l];
                break;
            }
        }
        if (!line) continue;

        // 5. Bir sonraki durağın indeksini hesapla
        int nextIndex = bus->currentStopIndex + bus->direction;
        if (nextIndex < 0 || nextIndex >= line->stopCount) {
            bus->direction *= -1;
            nextIndex = bus->currentStopIndex + bus->direction;
            if (nextIndex < 0) nextIndex = 0;
            if (nextIndex >= line->stopCount) nextIndex = line->stopCount - 1;
        }

        // 6. Hedef durağın koordinatlarını bul
        char stopChar = line->stops[nextIndex];
        int targetX = bus->posX, targetY = bus->posY;
        for (int s = 0; s < 16; s++) {
            if (busStops[s].name == stopChar) {
                targetX = busStops[s].x;
                targetY = busStops[s].y;
                break;
            }
        }

        // 7. Yol oluştur
        bus->pathLen = findPath(bus->posX, bus->posY, targetX, targetY, bus->pathX, bus->pathY);
        bus->pathIndex = 0;

        // 8. Durak indeksini güncelle
        bus->currentStopIndex = nextIndex;

        // 9. Eğer yol yoksa (aynı yerdeyse) hemen bekleme başlat
        if (bus->pathLen == 0) {
            bus->waitTime = 3;
        }
    }
}
