Лабораторная работа №6: Параллельная реализация метода сопряжённых градиентов с декартовой топологией 🚀
Автор: Ваше имяДата: Октябрь 2025
Цель работы 🎯
Реализовать и проанализировать параллельный алгоритм метода сопряжённых градиентов (CG) с использованием декартовой топологии в MPI. Исследовать эффективность и масштабируемость в сравнении с лабораторной работой №3 (базовый CG, полная и упрощённая версии).
Стек технологий 🛠️

Язык программирования: Python
Библиотеки: mpi4py, numpy, matplotlib
Реализация MPI: OpenMPI

Теоретическая часть 📚
Метод сопряжённых градиентов (CG) решает систему ( Ax = b ), эффективен для разреженных систем. В данной работе используется 2D декартова топология для оптимизации коммуникаций:

Декомпозиция данных: Матрица ( A ) делится по строкам, векторы ( x, r, p ) распределяются между процессами.
Топология: Тороидальная 2D-сетка упрощает обмен данными.
Коммуникации: Sendrecv_replace для локальных обменов, Allreduce для синхронизации.
Сравнение с ЛР3: Анализ производительности топологической реализации.

Часть 1: Создание декартовой топологии 🌐
Структура программы

Инициализация MPI, проверка квадратности числа процессов (( size = root^2 )).
Создание 2D тороидальной топологии (comm.Create_cart).
Определение соседей (Shift для вертикали и горизонтали).
Кольцевой обмен: передача рангов по горизонтали (Sendrecv_replace), суммирование в строке (row_comm.allreduce).
Вывод координат, соседей и сумм.

Код
cartesian_topology.py
from mpi4py import MPI
import numpy as np
import math

comm = MPI.COMM_WORLD
rank = comm.Get_rank()
size = comm.Get_size()

root = int(math.sqrt(size))
if root * root != size:
    if rank == 0:
        print("Ошибка: Число процессов должно быть квадратом натурального числа!")
    MPI.Finalize()
    exit()

dims = (root, root)
periods = (True, True)
comm_cart = comm.Create_cart(dims=dims, periods=periods, reorder=True)

new_rank = comm_cart.Get_rank()
coords = comm_cart.Get_coords(new_rank)

print(f"Процесс {rank} -> Новый ранг {new_rank}, Координаты {coords}")

neighbour_up, neighbour_down = comm_cart.Shift(direction=0, disp=1)
neighbour_left, neighbour_right = comm_cart.Shift(direction=1, disp=1)

print(f"Процесс {new_rank} (coords {coords}): Верхний сосед = {neighbour_up}, "
      f"Нижний сосед = {neighbour_down}, Левый сосед = {neighbour_left}, "
      f"Правый сосед = {neighbour_right}")

a = np.array([new_rank], dtype=np.float64)
total_sum = a.copy()

row_comm = comm_cart.Sub((False, True))

for _ in range(dims[1] - 1):
    send_buf = a.copy()
    comm_cart.Sendrecv_replace(send_buf, dest=neighbour_right, source=neighbour_left)
    total_sum += send_buf

row_sum = np.array([new_rank], dtype=np.float64)
row_sum = row_comm.allreduce(row_sum, op=MPI.SUM)

print(f"Процесс {new_rank} (coords {coords}): Сумма после кольца = {row_sum[0]}")

Верификация

Исправление ошибки: Завышенные суммы (2.0, 10.0; 9.0, 36.0, 63.0; 24.0, 88.0, 152.0, 216.0) устранены разделением обмена и суммирования.
Результаты:
4 процесса (2x2): строка 0 (0+1) = 1.0; строка 1 (2+3) = 5.0.
9 процессов (3x3): строка 0 (0+1+2) = 3.0; строка 1 (3+4+5) = 12.0; строка 2 (6+7+8) = 21.0.
16 процессов (4x4): строка 0 (0+1+2+3) = 6.0; строка 1 (4+5+6+7) = 22.0; строка 2 (8+9+10+11) = 38.0; строка 3 (12+13+14+15) = 54.0.



Часть 2: Метод CG с топологией 🔄
Структура программы

Инициализация MPI и топологии (( root \times root )).
Декомпозиция матрицы ( A ) (( n \times n )) по строкам (( n_{\text{local}} = n / root
