//программа-виселица, игра. версия 0.1. нужно угадывать слова по буквам - слова на латинице

#include <iostream>
#include <cstdlib>
#include <string>
#include <cstring>
#include <ctime>

using namespace std;

void Game(string* a);
bool Pic1(int n, int lenl, string* a);
void Pic2(int n);
char pic_mass[30];
int mas1[5][4] = { {1, 1, 1, 1}, {1, 256, 2, 256}, {1, 4, 3, 5}, {1, 256, 6, 256}, {1, 7, 256, 8} }; //целочисленный массив, в котором происходит сранвниене
char mas2[5][4] = { {'|', '-', 'T', '-'}, {'|', ' ', 'o', ' '}, {'|', '-', '|', '-'}, {'|', ' ', '^', ' '}, {'|', '/', ' ', '\\'} }; //массив символьный, в нём храниться картинка

//==================================================

int main()
{
    setlocale(LC_CTYPE, "");
    string sl0 = "paralelogramm"; //слова  - заменить русский на латиницу, так как на русском стринг и чар не сравниваются - ошибка кодировок
    string sl1 = "fizika";
    string sl2 = "kosinys";
    string sl3 = "program";
    string sl4 = "syshestvitelnoe";
    string sl5 = "domashka";
    string sl6 = "kollaider";
    string sl7 = "vosstanovlenie";
    string sl8 = "gipopotam";
    string sl9 = "torrent";

    string* mass[10]; //массив указателей на слова
    mass[0] = &sl0;
    mass[1] = &sl1;
    mass[2] = &sl2;
    mass[3] = &sl3;
    mass[4] = &sl4;
    mass[5] = &sl5;
    mass[6] = &sl6;
    mass[7] = &sl7;
    mass[8] = &sl8;
    mass[9] = &sl9;

    for (int i = 0; i < 30; i++)
    {
        pic_mass[i] = '_';
    }

    srand(time(NULL));
    int Random = rand() % 9;
    Game(mass[Random]);

    cin.sync();
    cin.clear();
    cin.get();

    return 0;
}

//==================================================

void Game(string* a) //функция - игра
{
    system("cls");
    int i = 0, j = 0, ochibka = 0;
    bool verno = false;
    int lenl = a->size(); // получить длину переданного слова
    char bukva;

    while (i <= 34) //пока в русском языке не кончились буквы
    {
        while (j != lenl)
        {
            cout << "_"; j++;
        } //первый раз выводим символ по количеству букв в слове

        setlocale(LC_CTYPE, "");
        cout << endl << "Назовите букву:" << endl;
        setlocale(LC_CTYPE, "");
        cin >> bukva;
        for (int k = 0; k < lenl; k++) //поиск 
        {
            setlocale(LC_CTYPE, "");
            if ((*a)[k] == bukva) // если совпадение есть, номер совпавшей буквы передаётся в первую рисующую функцию, которая эту букву "открывает"
            {
                verno = true;
                if (Pic1(k, lenl, a))
                {//проверка на то, было ли совпадение   
                    cout << endl << "Вы выиграли!";
                    return;
                }
            }
        }
        if (verno == false) //если совпадения не было
        {
            ochibka++;//число ошибок увеличивается на 1
            Pic2(ochibka); //передаётся как параметр в функцию, рисующую виселицу
            if (ochibka == 8)
            {
                setlocale(LC_CTYPE, "");
                cout << endl << "Вы проиграли!" << endl << "Правильный ответ: " << (*a);
                cin.sync();
                cin.clear();
                cin.get();
                return;
            }
        }

        i++; verno = false;
    }

}

//==================================================

bool Pic1(int n, int lenl, string* a) //n - номер буквы
{
    system("cls");
    setlocale(LC_CTYPE, "");
    pic_mass[n] = (*a)[n];
    for (int i = 0; i < lenl; i++)
    {
        cout << pic_mass[i];
    }

    for (int j = 0; j < lenl; j++)
    {
        if (pic_mass[j] == '_') return false;
    }
    return true;
};

//==================================================

void Pic2(int n) //n - число ошибок
{
    system("cls");
    for (int i = 0; i < 5; i++) //переключение по строкам
    {
        for (int j = 0; j < 4; j++)// переключение по столбцам
            if (mas1[i][j] <= n)
            {
                cout << mas2[i][j]; // вывести символ, стоящий в аналогичной ячейке второго массива
            }
            else
                cout << " ";  // вывести проблел
        cout << endl;
    }
}

//==================================================
