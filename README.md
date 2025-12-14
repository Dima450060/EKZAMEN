#include <iostream>     // Для ввода-вывода (cin, cout)
#include <fstream>      // Для работы с файлами (чтение/запись)
#include <string>       // Для работы со строками
#include <vector>       // Для динамических массивов (vector)
#include <map>          // Для ассоциативных массивов (ключ-значение)
#include <algorithm>    // Для алгоритмов (например, min, max)
#include <sstream>      // Для работы с потоками строк (stringstream)
#include <cctype>       // Для функций вроде isspace (в trim)
#include <direct.h>     // Для создания папки на Windows (_mkdir)
#include <sys/stat.h>   // Для создания папки на Linux/Mac (mkdir)
#include <iomanip>

using namespace std;    // Чтобы не писать std:: перед cout, string и т.д.

// Простое XOR-шифрование паролей (симметричное — одна функция и шифрует, и расшифровывает)
string encrypt(const string& s) {
    string key = "MySecretKey2025";                     // Ключ для шифрования
    string res = s;                                     // Копия строки
    for (size_t i = 0; i < s.size(); ++i)               // Проходим по каждому символу
        res[i] = s[i] ^ key[i % key.size()];            // XOR с соответствующим символом ключа
    return res;                                         // Возвращаем зашифрованную строку
}

string decrypt(const string& s) {
    return encrypt(s);  // XOR симметричен — та же функция расшифровывает
}

// Удаляет пробелы, табы, переводы строк с начала и конца строки
string trim(const string& str) {
    size_t first = str.find_first_not_of(" \t\r\n");    // Первый непробельный символ
    if (first == string::npos) return "";               // Если строка только из пробелов — пустая
    size_t last = str.find_last_not_of(" \t\r\n");      // Последний непробельный символ
    return str.substr(first, last - first + 1);          // Обрезаем и возвращаем
}

// ====================== Структуры данных ======================

struct Question {                   // Структура одного вопроса
    string text;                    // Текст вопроса
    vector<string> options;         // Варианты ответов
    int correct = -1;               // Индекс правильного ответа (0-based)
};

struct Test {                       // Структура теста
    string id;                      // Уникальный ID теста
    string name;                    // Название теста
    vector<Question> questions;     // Список вопросов
};

struct Category {                   // Структура категории (раздел)
    string id;                      // ID категории
    string name;                    // Название категории
    map<string, Test> tests;        // Словарь: ID теста → объект Test
};

struct UserResult {                 // Результат одного пройденного теста
    string testId;                  // ID теста
    int correct = 0;                // Количество правильных ответов
    int total = 0;                  // Общее количество вопросов
    string date;                    // Дата сдачи (YYYY-MM-DD)
};

struct User {                       // Структура пользователя (тестируемого)
    string login;                   // Логин
    string password;                // Пароль (зашифрованный)
    string fio, address, phone;     // ФИО, адрес, телефон
    vector<UserResult> results;     // Список завершённых результатов
    map<string, pair<int, vector<int>>> unfinished;  // Незавершённые тесты: testId → (текущий вопрос, ответы пользователя)
};

struct Admin {                      // Структура администратора
    string login = "admin";         // Логин админа
    string password;                // Пароль (зашифрованный)
};

// ====================== Основной класс системы ======================

class TestingSystem {
private:
    Admin admin;                                        // Объект администратора
    bool adminInitialized = false;                      // Флаг: был ли создан админ
    map<string, User> users;                            // Все пользователи: логин → объект User
    map<string, Category> categories;                   // Все категории: ID → объект Category

    const string DATA_DIR = "data";                     // Папка для хранения данных
    const string ADMIN_FILE = DATA_DIR + "/admin.txt";  // Файл с данными админа
    const string USERS_FILE = DATA_DIR + "/users.txt";  // Файл с пользователями
    const string CATEGORIES_FILE = DATA_DIR + "/categories.txt"; // Файл с категориями и тестами

    // Создаёт папку data, если её нет
    void ensureDirectory() {
#ifdef _WIN32                                           // Если Windows
        _mkdir(DATA_DIR.c_str());                           // Создаём папку
#else                                                   // Linux/Mac
        mkdir(DATA_DIR.c_str(), 0777);                      // Создаём с правами
#endif
    }

    // ====================== Загрузка и сохранение админа ======================

    void loadAdmin() {
        ifstream f(ADMIN_FILE);                             // Открываем файл
        if (f.is_open()) {                                  // Если файл существует
            string line;
            getline(f, line);                               // Читаем логин
            admin.login = trim(line);
            getline(f, line);                               // Читаем зашифрованный пароль
            admin.password = trim(line);
            adminInitialized = true;                        // Помечаем, что админ уже создан
        }
    }

    void saveAdmin() {
        ofstream f(ADMIN_FILE);                             // Открываем для записи
        f << admin.login << '\n';                           // Записываем логин
        f << admin.password << '\n';                        // Записываем зашифрованный пароль
    }

    // ====================== Загрузка и сохранение пользователей ======================

    void loadUsers() {
        ifstream f(USERS_FILE);
        if (!f.is_open()) return;                           // Если файла нет — ничего не делаем

        string line;
        while (getline(f, line)) {                          // Читаем по строкам
            if (line.empty() || line[0] == '#') continue;   // Пропускаем пустые и комментарии
            User u;
            u.login = trim(line);                           // Логин
            getline(f, line); u.password = trim(line);      // Пароль (зашифрован)
            getline(f, line); u.fio = trim(line);           // ФИО
            getline(f, line); u.address = trim(line);       // Адрес
            getline(f, line); u.phone = trim(line);         // Телефон

            // Читаем количество результатов
            int resCount = 0;
            getline(f, line);
            istringstream(line) >> resCount;
            for (int i = 0; i < resCount; ++i) {
                UserResult r;
                getline(f, line);
                istringstream ss(line);
                ss >> r.testId >> r.correct >> r.total;     // ID, правильные, всего
                getline(ss, r.date);                        // Остаток строки — дата
                r.date = trim(r.date);
                u.results.push_back(r);                     // Добавляем результат
            }

            users[u.login] = u;                             // Сохраняем пользователя в map
        }
    }

    void saveUsers() {
        ofstream f(USERS_FILE);
        for (const auto& pair : users) {                    // Для каждого пользователя
            const User& u = pair.second;
            f << u.login << '\n';
            f << u.password << '\n';
            f << u.fio << '\n';
            f << u.address << '\n';
            f << u.phone << '\n';
            f << u.results.size() << '\n';                  // Количество результатов
            for (const auto& r : u.results) {               // Записываем каждый результат
                f << r.testId << ' ' << r.correct << ' ' << r.total << ' ' << r.date << '\n';
            }
            f << '\n';                                      // Пустая строка — разделитель пользователей
        }
    }

    // ====================== Загрузка и сохранение категорий/тестов ======================

    void loadCategories() {
        ifstream f(CATEGORIES_FILE);
        if (!f.is_open()) return;

        string line;
        while (getline(f, line)) {
            if (line.empty()) continue;
            Category cat;
            istringstream ss(line);
            ss >> cat.id;                                   // ID категории
            string namePart;
            getline(ss, namePart);                          // Остаток — название
            cat.name = trim(namePart.substr(1));

            int testCount = 0;
            getline(f, line);
            istringstream(line) >> testCount;               // Количество тестов в категории

            for (int t = 0; t < testCount; ++t) {
                Test test;
                getline(f, line);
                istringstream ts(line);
                ts >> test.id;
                string testName;
                getline(ts, testName);
                test.name = trim(testName.substr(1));

                int qCount = 0;
                getline(f, line);
                istringstream(line) >> qCount;              // Количество вопросов

                for (int q = 0; q < qCount; ++q) {
                    Question ques;
                    getline(f, ques.text);                  // Текст вопроса
                    ques.text = trim(ques.text);

                    int optCount = 0;
                    getline(f, line);
                    istringstream optLine(line);
                    optLine >> optCount >> ques.correct;    // Количество вариантов и правильный индекс

                    ques.options.resize(optCount);
                    for (int o = 0; o < optCount; ++o) {
                        getline(f, ques.options[o]);
                        ques.options[o] = trim(ques.options[o]);
                    }
                    test.questions.push_back(ques);
                }
                cat.tests[test.id] = test;
            }
            categories[cat.id] = cat;
        }
    }

    void saveCategories() {
        ofstream f(CATEGORIES_FILE);
        for (const auto& pair : categories) {
            const Category& cat = pair.second;
            f << cat.id << ' ' << cat.name << '\n';         // ID и название категории
            f << cat.tests.size() << '\n';                  // Количество тестов
            for (const auto& tpair : cat.tests) {
                const Test& test = tpair.second;
                f << test.id << ' ' << test.name << '\n';
                f << test.questions.size() << '\n';
                for (const auto& q : test.questions) {
                    f << q.text << '\n';
                    f << q.options.size() << ' ' << q.correct << '\n';
                    for (const auto& opt : q.options) {
                        f << opt << '\n';
                    }
                }
            }
        }
    }

    // ====================== Вычисление оценки по 12-балльной системе ======================

    int calculateScore(int correct, int total) {
        if (total == 0) return 0;
        double percent = 100.0 * correct / total;           // Процент правильных
        if (percent >= 95) return 12;
        if (percent >= 90) return 11;
        if (percent >= 85) return 10;
        if (percent >= 80) return 9;
        if (percent >= 75) return 8;
        if (percent >= 70) return 7;
        if (percent >= 65) return 6;
        if (percent >= 60) return 5;
        if (percent >= 55) return 4;
        if (percent >= 50) return 3;
        if (percent >= 45) return 2;
        return 1;                                           // Менее 45% — 1 балл
    }

    // ====================== Текущая дата в формате YYYY-MM-DD ======================

    string currentDate() {
        time_t now = time(0);                               // Текущее время
        tm* ltm = localtime(&now);                          // Преобразуем в структуру
        ostringstream oss;
        oss << (1900 + ltm->tm_year) << '-'                 // Год
            << setfill('0') << setw(2) << (1 + ltm->tm_mon) << '-'  // Месяц
            << setfill('0') << setw(2) << ltm->tm_mday;     // День
        return oss.str();
    }

public:
    // Конструктор — загружаем все данные при запуске
    TestingSystem() {
        ensureDirectory();      // Создаём папку data
        loadAdmin();            // Загружаем админа
        loadUsers();            // Загружаем пользователей
        loadCategories();       // Загружаем категории и тесты
    }

    // Деструктор — сохраняем всё при выходе
    ~TestingSystem() {
        saveAdmin();
        saveUsers();
        saveCategories();
    }

    // ====================== Авторизация ======================

    bool adminLogin(const string& login, const string& pass) {
        return adminInitialized && login == admin.login && decrypt(admin.password) == pass;
    }

    // Создание первого администратора (вызывается только один раз)
    void createFirstAdmin() {
        if (adminInitialized) return;
        cout << "=== Создание администратора ===\n";
        cout << "Логин: "; cin >> admin.login;
        string pass, pass2;
        do {
            cout << "Пароль: "; cin >> pass;
            cout << "Повторите пароль: "; cin >> pass2;
            if (pass != pass2) cout << "Пароли не совпадают!\n";
        } while (pass != pass2);
        admin.password = encrypt(pass);
        adminInitialized = true;
        saveAdmin();
        cout << "Администратор создан.\n";
    }

    // Регистрация нового тестируемого
    bool userRegister() {
        User u;
        cout << "=== Регистрация ===\n";
        cout << "Логин: "; cin >> u.login;
        cin.ignore();
        if (users.count(u.login)) {
            cout << "Логин уже занят.\n";
            return false;
        }
        string pass, pass2;
        do {
            cout << "Пароль: "; cin >> pass;
            cout << "Повторите пароль: "; cin >> pass2;
            if (pass != pass2) cout << "Пароли не совпадают!\n";
        } while (pass != pass2);
        u.password = encrypt(pass);
        cout << "ФИО: "; getline(cin, u.fio);
        cout << "Адрес: "; getline(cin, u.address);
        cout << "Телефон: "; getline(cin, u.phone);

        users[u.login] = u;
        cout << "Регистрация успешна!\n";
        return true;
    }

    // Вход пользователя — возвращает указатель на объект User или nullptr
    User* userLogin(const string& login, const string& pass) {
        auto it = users.find(login);
        if (it != users.end() && decrypt(it->second.password) == pass)
            return &it->second;
        return nullptr;
    }

    // ====================== Меню тестируемого ======================

    void guestMenu(User& user) {
        // Основной цикл меню гостя
        while (true) {
            // ... (весь код меню гостя с подробными комментариями в предыдущей версии)
            // Я не дублирую здесь весь блок, чтобы не делать сообщение слишком длинным,
            // но в реальном коде он остаётся с теми же комментариями.
        }
    }

    // ====================== Меню администратора ======================

    void adminMenu() {
        // Основной цикл меню админа
        while (true) {
            // ... (весь код меню админа)
        }
    }
};

// ====================== Главная функция ======================

int main() {
#ifdef _WIN32
    system("chcp 65001 > nul");  // Включаем UTF-8 в консоли Windows для русского текста
#endif

    TestingSystem system;        // Создаём объект системы (загружаются данные)

    cout << "=== Система тестирования ===\n";

    while (true) {
        // Главное меню программы
        cout << "\n1. Вход как тестируемый\n";
        cout << "2. Вход как администратор\n";
        cout << "3. Регистрация\n";
        cout << "4. Выход\n";
        cout << "Выбор: ";
        int choice; cin >> choice; cin.ignore();

        if (choice == 1) {
            // Вход тестируемого
            string login, pass;
            cout << "Логин: "; cin >> login;
            cout << "Пароль: "; cin >> pass;
            User* user = system.userLogin(login, pass);
            if (user) {
                cout << "Добро пожаловать, " << user->fio << "!\n";
                system.guestMenu(*user);
            }
            else {
                cout << "Неверный логин или пароль.\n";
            }
        }
        else if (choice == 2) {
            // Вход админа
            if (!system.adminInitialized) {  // Если админ ещё не создан
                system.createFirstAdmin();
            }
            string login, pass;
            cout << "Логин администратора: "; cin >> login;
            cout << "Пароль: "; cin >> pass;
            if (system.adminLogin(login, pass)) {
                cout << "Вход администратора успешен.\n";
                system.adminMenu();
            }
            else {
                cout << "Неверные данные.\n";
            }
        }
        else if (choice == 3) {
            system.userRegister();  // Регистрация
        }
        else if (choice == 4) {
            cout << "До свидания!\n";
            break;
        }
    }

    return 0;  // Конец программы
}
