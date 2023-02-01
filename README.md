from random import randint


class Dot:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __repr__(self):
        return f"({self.x}, {self.y})"


class BoardException(Exception):
    pass


class BoardOutException(BoardException):
    def __str__(self):
        return "Вы пытаетесь выстрелить за доску!"


class BoardUsedException(BoardException):
    def __str__(self):
        return "Вы уже стреляли в эту клетку"


class BoardWrongShipException(BoardException):
    pass


class Ship:
    def __init__(self, bow, l, o):
        self.bow = bow
        self.l = l
        self.o = o
        self.lives = l

    @property
    def dots(self):
        ship_dots = []
        for i in range(self.l):
            cur_x = self.bow.x
            cur_y = self.bow.y

            if self.o == 0:
                cur_x += i

            elif self.o == 1:
                cur_y += i

            ship_dots.append(Dot(cur_x, cur_y))

        return ship_dots

    def shooten(self, shot):
        return shot in self.dots


class Board:
    def __init__(self, hid=False, size=6):
        self.size = size
        self.hid = hid

        self.count = 0

        self.field = [["O"] * size for _ in range(size)]

        self.busy = []
        self.ships = []

    def add_ship(self, ship):

        for d in ship.dots:
            if self.out(d) or d in self.busy:
                raise BoardWrongShipException()
        for d in ship.dots:
            self.field[d.x][d.y] = "■"
            self.busy.append(d)

        self.ships.append(ship)
        self.contour(ship)

    def contour(self, ship, verb=False):
        near = [
            (-1, -1), (-1, 0), (-1, 1),
            (0, -1), (0, 0), (0, 1),
            (1, -1), (1, 0), (1, 1)
        ]
        for d in ship.dots:
            for dx, dy in near:
                cur = Dot(d.x + dx, d.y + dy)
                if not (self.out(cur)) and cur not in self.busy:
                    if verb:
                        self.field[cur.x][cur.y] = "*"
                    self.busy.append(cur)

    def __str__(self):
        res = ""
        res += "  | a | b | c | d | e | f |"
        for i, row in enumerate(self.field):
            res += f"\n{i + 1} | " + " | ".join(row) + " |"

        if self.hid:
            res = res.replace("■", "O")
        return res

    def out(self, d):
        return not ((0 <= d.x < self.size) and (0 <= d.y < self.size))

    def shot(self, d):
        if self.out(d):
            raise BoardOutException()

        if d in self.busy:
            raise BoardUsedException()

        self.busy.append(d)

        for ship in self.ships:
            if d in ship.dots:
                ship.lives -= 1
                self.field[d.x][d.y] = "X"
                if ship.lives == 0:
                    self.count += 1
                    self.contour(ship, verb=True)
                    print("     Корабль уничтожен!")
                    return False
                else:
                    print("     Корабль подбит!!")
                    return True

        self.field[d.x][d.y] = "*"
        print("     Промах!")
        return False

    def begin(self):
        self.busy = []


class Player:
    def __init__(self, board, enemy):
        self.board = board
        self.enemy = enemy
        self.letter = {'a': 0, 'b': 1, 'c': 2, 'd': 3, 'e': 4, 'f': 5}
        self.un_letter = {0: 'a', 1: 'b', 2: 'c', 3: 'd', 4: 'e', 5: 'f'}

    def ask(self):
        raise NotImplementedError()

    def move(self):
        while True:
            try:
                target = self.ask()
                repeat = self.enemy.shot(target)
                return repeat
            except BoardException as e:
                print(e)


class AI(Player):
    def ask(self):
        ft = Board()
        d = Dot(randint(0, 5), randint(0, 5))
        if d in ft.busy:
            d = Dot(d.x, (d.y + randint(-1, 1))) or Dot((d.x + randint(-1, 1)), d.y)
        print(f"Ход компьютера: {self.un_letter[d.y]} {d.x + 1}")
        return d


class User(Player):
    def ask(self):
        while True:
            cord_x = input("Введите букву столбца: ")

            if cord_x not in self.letter:
                print("Вы ввели столбец вне диапазона поля!")
                continue
            x = self.letter[cord_x]

            cord_y = input("Введите номер строки: ")
            y = cord_y
            if not (y.isdigit()):
                print("Введите числа! ")
                continue

            x, y = int(x), int(y)

            return Dot(y - 1, x)


class Game:
    def __init__(self, size=6):
        self.size = size
        pl = self.random_board()
        co = self.random_board()
        co.hid = True

        self.ai = AI(co, pl)
        self.us = User(pl, co)



    def random_place(self):
        lens = [3, 2, 2, 1, 1, 1, 1]
        board = Board(size=self.size)
        attempts = 0
        for l in lens:
            while True:
                attempts += 1
                if attempts > 2000:
                    return None
                ship = Ship(Dot(randint(0, self.size), randint(0, self.size)), l, randint(0, 1))
                try:
                    board.add_ship(ship)
                    break
                except BoardWrongShipException:
                    pass
        board.begin()
        return board

    def random_board(self):
        board = None
        while board is None:
            board = self.random_place()
        return board

    def greet(self):
        print("-" * 38)
        print('Добро пожаловать в игру "Морской Бой"')
        print("-" * 38)
        print("Правила: поочередно вводите координаты"
              "\nстолбца и строки.""\n")

    def loop(self):
        num = 0
        self.shipst = Board
        while True:
            print(" " * 3,"-" * 22)
            print("     Доска пользователя:")
            print(self.us.board)
            print(" " * 3,"-" * 22)
            print("      Доска компьютера:")
            print(self.ai.board)
            if num % 2 == 0:
                print(" " * 3,"-" * 22)
                print("     Ходит компьютер!")
                repeat = self.ai.move()
            else:
                print(" " * 3,"-" * 22)
                print("     Ходит пользователь!""\n")
                repeat = self.us.move()
            if repeat:
                num -= 1

            if self.ai.board.count == 7:
                print("-" * 20)
                print("Пользователь выиграл!")
                print(self.us.board)
                break

            if self.us.board.count == 7:
                print("-" * 20)
                print("Компьютер выиграл!")
                print(self.ai.board)
                break
            num += 1

    def start(self):
        self.greet()
        self.loop()


g = Game()
g.start()
