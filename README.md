# pylint: disable=no-name-in-module
import sys
import random as r
from PyQt5.QtWidgets import QMainWindow, QFrame, QDesktopWidget, QApplication, QLabel
from PyQt5.QtCore import Qt, QBasicTimer, QTimer
from PyQt5.QtGui import QPainter, QColor, QFont, QBrush, QPen


class Segment:
    SegmentSize = 20  # Уменьшил для более плавного движения

    def __init__(self, x, y, is_head=False):
        self.x = x
        self.y = y
        self.is_head = is_head
        self.set_color()

    def set_color(self):
        if self.is_head:
            self.color = QColor(34, 139, 34)  # Зеленый для головы
        else:
            self.color = QColor(0, 128, 0)  # Темно-зеленый для тела

    def drawSegment(self, painter):
        x1, y1, x2, y2 = self.coords()

        # Рисуем сегмент с закругленными краями
        painter.setBrush(QBrush(self.color))
        painter.setPen(Qt.NoPen)
        painter.drawRoundedRect(x1 + 1, y1 + 1, 
                               Segment.SegmentSize - 2, 
                               Segment.SegmentSize - 2, 
                               5, 5)

        # Эффект объема для головы
        if self.is_head:
            painter.setPen(QPen(QColor(144, 238, 144), 2))
            painter.drawRoundedRect(x1 + 3, y1 + 3, 
                                   Segment.SegmentSize - 6, 
                                   Segment.SegmentSize - 6, 
                                   3, 3)
            # Глаза змейки
            painter.setBrush(QBrush(QColor(255, 255, 255)))
            painter.drawEllipse(x1 + 5, y1 + 5, 4, 4)
            painter.drawEllipse(x1 + 11, y1 + 5, 4, 4)

    def coords(self):
        return (self.x, self.y, 
                self.x + Segment.SegmentSize, 
                self.y + Segment.SegmentSize)


class Snake:
    def __init__(self, segments):
        self.segments = segments
        self.segments[0].is_head = True  # Первый сегмент - голова
        self.mapping = {
            "Down": (0, 1),
            "Right": (1, 0),
            "Up": (0, -1),
            "Left": (-1, 0)
        }
        self.vector = self.mapping["Right"]
        self.score = 0
        self.last_move = "Right"

    def move(self):
        # Обновляем статус головы
        for seg in self.segments:
            seg.is_head = False
        
        # Двигаем сегменты
        for i in range(len(self.segments) - 1, 0, -1):
            self.segments[i].x = self.segments[i - 1].x
            self.segments[i].y = self.segments[i - 1].y
        
        # Двигаем голову
        self.segments[0].x += Segment.SegmentSize * self.vector[0]
        self.segments[0].y += Segment.SegmentSize * self.vector[1]
        self.segments[0].is_head = True

    def addSegment(self):
        last_seg = self.segments[-1]
        new_seg = Segment(last_seg.x, last_seg.y)
        self.segments.append(new_seg)
        self.score += 10  # Очки за съеденное яблоко

    def setDirection(self, direction):
        # Предотвращаем разворот на 180 градусов
        opposite_directions = {
            "Up": "Down",
            "Down": "Up",
            "Left": "Right",
            "Right": "Left"
        }
        
        if self.last_move == opposite_directions.get(direction):
            return
            
        if direction in self.mapping:
            self.vector = self.mapping[direction]
            self.last_move = direction


class Apple(Segment):
    def __init__(self, x, y):
        super().__init__(x, y)
        self.color = QColor(255, 69, 0)  # Красный для яблока
    
    def drawSegment(self, painter):
        x1, y1, x2, y2 = self.coords()
        
        # Рисуем яблоко
        painter.setBrush(QBrush(self.color))
        painter.setPen(Qt.NoPen)
        painter.drawEllipse(x1 + 2, y1 + 2, 
                           Segment.SegmentSize - 4, 
                           Segment.SegmentSize - 4)
        
        # Черенок яблока
        painter.setPen(QPen(QColor(139, 69, 19), 2))
        painter.drawLine(x1 + Segment.SegmentSize//2, 
                        y1 + 2, 
                        x1 + Segment.SegmentSize//2, 
                        y1 - 3)
        
        # Блеск на яблоке
        painter.setBrush(QBrush(QColor(255, 255, 255, 150)))
        painter.drawEllipse(x1 + 5, y1 + 5, 4, 4)


class GameWindow(QMainWindow):
    Width = 800
    Height = 600

    def __init__(self):
        super().__init__()
        self.score_label = None
        self.initUI()

    def initUI(self):
        self.board = Board(self)
        self.board.setGeometry(0, 30, GameWindow.Width, GameWindow.Height - 30)

        # Создаем панель счета
        self.score_panel = QLabel(self)
        self.score_panel.setGeometry(10, 5, 200, 25)
        self.score_panel.setStyleSheet("""
            QLabel {
                color: #2E8B57;
                font-weight: bold;
                font-size: 14px;
            }
        """)

        self.resize(GameWindow.Width, GameWindow.Height)
        self.center()
        self.setWindowTitle('🐍 Питончик - Классическая Змейка')
        
        # Устанавливаем иконку окна
        self.setStyleSheet("""
            QMainWindow {
                background-color: #F0F8FF;
            }
        """)
        
        self.show()

    def center(self):
        screen = QDesktopWidget().screenGeometry()
        size = self.geometry()
        self.move(int((screen.width() - size.width()) / 2),
                  int((screen.height() - size.height()) / 2))

    def updateScore(self, score):
        self.score_panel.setText(f"Счёт: {score} | Длина: {score//10 + 3}")


class Board(QFrame):
    def __init__(self, parent):
        super().__init__(parent)
        self.timer = QBasicTimer()
        self.game_timer = QTimer()
        self.game_time = 0
        
        self.setFocusPolicy(Qt.StrongFocus)
        self.isStarted = False
        self.isPaused = False
        self.start()

    def addApple(self):
        # Генерация яблока в случайном месте
        max_x = (GameWindow.Width // Segment.SegmentSize) - 1
        max_y = ((GameWindow.Height - 30) // Segment.SegmentSize) - 1
        
        while True:
            x = r.randint(0, max_x) * Segment.SegmentSize
            y = r.randint(0, max_y) * Segment.SegmentSize
            
            # Проверяем, не попадает ли яблоко на змейку
            collision = False
            for segment in self.snake.segments:
                if segment.x == x and segment.y == y:
                    collision = True
                    break
            
            if not collision:
                return Apple(x, y)

    def start(self):
        self.speed = 150
        self.snake = Snake([Segment(60, 20), Segment(40, 20), Segment(20, 20)])
        self.apple = self.addApple()
        self.isPaused = False
        self.isStarted = True
        
        # Запускаем таймеры
        self.timer.start(self.speed, self)
        self.game_timer.timeout.connect(self.updateGameTime)
        self.game_timer.start(1000)  # Обновляем каждую секунду
        
        self.update()

    def pause(self):
        if not self.isStarted:
            return

        self.isPaused = not self.isPaused

        if self.isPaused:
            self.timer.stop()
            self.game_timer.stop()
        else:
            self.timer.start(self.speed, self)
            self.game_timer.start(1000)

        self.update()

    def stop(self):
        self.isStarted = False
        self.timer.stop()
        self.game_timer.stop()
        self.update()

    def updateGameTime(self):
        self.game_time += 1

    def paintEvent(self, event):
        # Фон с градиентом
        painter = QPainter(self)
        painter.setBrush(QBrush(QColor(240, 248, 255)))
        painter.setPen(Qt.NoPen)
        painter.drawRect(self.rect())
        
        # Сетка игрового поля
        painter.setPen(QPen(QColor(220, 220, 220, 50), 1))
        for x in range(0, GameWindow.Width, Segment.SegmentSize):
            painter.drawLine(x, 0, x, GameWindow.Height)
        for y in range(0, GameWindow.Height, Segment.SegmentSize):
            painter.drawLine(0, y, GameWindow.Width, y)

        # Рисуем игровые объекты
        self.apple.drawSegment(painter)
        for seg in self.snake.segments:
            seg.drawSegment(painter)

        # Сообщения
        if self.isPaused:
            self.drawMessage(painter, "⏸️ ПАУЗА", QColor(255, 140, 0))
        elif not self.isStarted:
            self.drawMessage(painter, "💀 ИГРА ОКОНЧЕНА", QColor(220, 20, 60))
            self.drawStats(painter)

        painter.end()

    def drawMessage(self, painter, text, color):
        painter.setPen(QPen(color, 2))
        painter.setFont(QFont('Arial Rounded MT Bold', 36))
        painter.drawText(self.rect(), Qt.AlignCenter, text)
        
        # Подзаголовок
        painter.setPen(QPen(QColor(105, 105, 105)))
        painter.setFont(QFont('Arial', 16))
        painter.drawText(self.rect(), Qt.AlignCenter | Qt.AlignBottom, 
                        "Нажмите N для новой игры")

    def drawStats(self, painter):
        stats = f"Финальный счёт: {self.snake.score} | Время: {self.game_time} сек."
        painter.setPen(QPen(QColor(47, 79, 79)))
        painter.setFont(QFont('Arial', 14))
        painter.drawText(self.rect(), Qt.AlignBottom | Qt.AlignHCenter, 
                        stats)

    def timerEvent(self, event):
        if event.timerId() == self.timer.timerId():
            self.snake.move()
            
            # Получаем координаты головы
            head = self.snake.segments[0]
            head_x, head_y = head.x, head.y
            
            # Проверка столкновения со стенами
            if (head_x < 0 or head_x >= GameWindow.Width or
                head_y < 0 or head_y >= GameWindow.Height - 30):
                self.stop()
                return
            
            # Проверка съедания яблока
            if head_x == self.apple.x and head_y == self.apple.y:
                self.snake.addSegment()
                self.apple = self.addApple()
                # Увеличиваем скорость
                if self.speed > 80:
                    self.speed -= 2
                    self.timer.start(self.speed, self)
                
                # Обновляем счет
                self.parent().updateScore(self.snake.score)
            
            # Проверка столкновения с собой
            for segment in self.snake.segments[1:]:
                if head_x == segment.x and head_y == segment.y:
                    self.stop()
                    return
                    
            self.update()

    def keyPressEvent(self, event):
        key = event.key()
        
        if key == Qt.Key_Down or key == Qt.Key_S:
            self.snake.setDirection("Down")
        elif key == Qt.Key_Up or key == Qt.Key_W:
            self.snake.setDirection("Up")
        elif key == Qt.Key_Left or key == Qt.Key_A:
            self.snake.setDirection("Left")
        elif key == Qt.Key_Right or key == Qt.Key_D:
            self.snake.setDirection("Right")
        elif key == Qt.Key_N:
            self.start()
            self.parent().updateScore(0)
        elif key == Qt.Key_P or key == Qt.Key_Space:
            self.pause()
        elif key == Qt.Key_Escape:
            self.parent().close()
        else:
            super().keyPressEvent(event)


if __name__ == '__main__':
    app = QApplication(sys.argv)
    
    # Устанавливаем стиль приложения
    app.setStyle('Fusion')
    
    game = GameWindow()
    
    # Показываем инструкцию при запуске
    from PyQt5.QtWidgets import QMessageBox
    msg = QMessageBox()
    msg.setWindowTitle("🐍 Управление в игре")
    msg.setText("""Управление змейкой:
    
Стрелки - Движение
Пробел - Пауза
N - Новая игра
ESC - Выход

Соберите как можно больше яблок!
Избегайте стен и себя самого!""")
    msg.setIcon(QMessageBox.Information)
    msg.exec_()
    
    sys.exit(app.exec_())
