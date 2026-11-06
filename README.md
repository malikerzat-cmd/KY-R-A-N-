import pygame
import random
import os
import sys

pygame.init()

# ========== GAME SETTINGS ==========

WIDTH = 600
HEIGHT = 720
CELL = 60
COLS = 10
FPS = 60

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("QYRAN - Жолдан өту")

clock = pygame.time.Clock()

WHITE = (255, 255, 255)
BLACK = (20, 20, 20)
GREEN = (50, 150, 65)
DARK_GREEN = (30, 100, 40)
ROAD = (55, 55, 60)
BLUE = (30, 120, 255)
RED = (220, 40, 40)
ORANGE = (240, 140, 20)
YELLOW = (255, 215, 0)

font = pygame.font.SysFont("arial", 25, bold=True)
big_font = pygame.font.SysFont("arial", 50, bold=True)
small_font = pygame.font.SysFont("arial", 19)

RECORD_FILE = "record.txt"


def load_record():
    if os.path.exists(RECORD_FILE):
        try:
            with open(RECORD_FILE) as f:
                return int(f.read())
        except ValueError:
            return 0
    return 0


def save_record(value):
    with open(RECORD_FILE, "w") as f:
        f.write(str(value))


# =====================================
# PLAYER
# =====================================

class Player:

    def __init__(self):
        self.reset()

    def reset(self):
        self.x = 4
        self.y = 0
        self.isAlive = True
        self.score = 0

    def move(self, direction):

        if not self.isAlive:
            return

        if direction == "up":
            self.y += 1

        elif direction == "down":
            self.y = max(0, self.y - 1)

        elif direction == "left":
            self.x -= 1

        elif direction == "right":
            self.x += 1

        self.x = max(0, min(COLS - 1, self.x))

    def draw(self, camera):

        y = HEIGHT - 120 - (self.y - camera) * CELL

        pygame.draw.rect(
            screen, BLUE,
            (self.x * CELL + 10, y + 10, 40, 40),
            border_radius=10
        )

        pygame.draw.circle(
            screen, WHITE,
            (self.x * CELL + 22, y + 23), 4
        )

        pygame.draw.circle(
            screen, WHITE,
            (self.x * CELL + 38, y + 23), 4
        )


# =====================================
# VEHICLE PARENT CLASS
# =====================================

class Vehicle:

    def __init__(self, x, lane, speed, direction, color, width):

        self.x = float(x)
        self.lane = lane
        self.speed = speed
        self.direction = direction
        self.color = color
        self.width = width

    def update(self, dt):

        self.x += self.speed * self.direction * dt

        if self.direction == 1 and self.x > WIDTH:
            self.respawn()

        elif self.direction == -1 and self.x + self.width < 0:
            self.respawn()

    def respawn(self):

        if self.direction == 1:
            self.x = -self.width - random.randint(50, 180)
        else:
            self.x = WIDTH + random.randint(50, 180)

    def get_rect(self, camera):

        y = HEIGHT - 120 - (self.lane - camera) * CELL

        return pygame.Rect(
            int(self.x), int(y + 13),
            self.width, 34
        )

    def draw(self, camera):

        rect = self.get_rect(camera)

        if -50 < rect.y < HEIGHT + 50:

            pygame.draw.rect(
                screen, self.color, rect,
                border_radius=8
            )


# =====================================
# CAR - INHERITANCE
# =====================================

class Car(Vehicle):

    def __init__(self, lane, direction, level):

        super().__init__(
            random.randint(0, WIDTH),
            lane,
            random.randint(100, 150) + level * 15,
            direction,
            RED,
            48
        )

    def draw(self, camera):

        super().draw(camera)

        rect = self.get_rect(camera)

        if -50 < rect.y < HEIGHT + 50:
            pygame.draw.rect(
                screen, (130, 200, 240),
                (rect.x + 12, rect.y + 5, 24, 12),
                border_radius=3
            )


# =====================================
# TRUCK - INHERITANCE
# =====================================

class Truck(Vehicle):

    def __init__(self, lane, direction, level):

        super().__init__(
            random.randint(0, WIDTH),
            lane,
            random.randint(70, 110) + level * 10,
            direction,
            ORANGE,
            75
        )


# =====================================
# ROAD LANE
# =====================================

class RoadLane:

    def __init__(self, number):

        self.number = number
        self.isRoad = number > 0 and number % 2 == 0

    def draw(self, camera):

        y = HEIGHT - 120 - (self.number - camera) * CELL

        if y < -CELL or y > HEIGHT:
            return

        if self.isRoad:

            pygame.draw.rect(
                screen, ROAD,
                (0, y, WIDTH, CELL)
            )

            for x in range(0, WIDTH, 90):

                pygame.draw.rect(
                    screen, WHITE,
                    (x, y + 29, 40, 3)
                )

        else:

            pygame.draw.rect(
                screen, GREEN,
                (0, y, WIDTH, CELL)
            )

            for x in range(20, WIDTH, 80):

                pygame.draw.circle(
                    screen, DARK_GREEN,
                    (x, y + 20), 4
                )


# =====================================
# COIN
# =====================================

class Coin:

    def __init__(self, lane):

        self.x = random.randint(0, COLS - 1)
        self.lane = lane
        self.isCollected = False
        self.value = 1

    def draw(self, camera):

        if self.isCollected:
            return

        y = HEIGHT - 120 - (self.lane - camera) * CELL

        if -30 < y < HEIGHT + 30:

            pygame.draw.circle(
                screen, YELLOW,
                (self.x * CELL + 30, int(y + 30)),
                15
            )

            pygame.draw.circle(
                screen, ORANGE,
                (self.x * CELL + 30, int(y + 30)),
                10, 2
            )


# =====================================
# GAME MANAGER
# =====================================

class GameManager:

    def __init__(self):

        self.record = load_record()
        self.startGame()

    def startGame(self):

        self.player = Player()

        self.score = 0
        self.level = 1
        self.camera = 0
        self.gameOver = False

        self.lanes = []
        self.vehicles = []
        self.coins = []

        self.last_safe_lane = 0

        self.lanes.append(RoadLane(0))

        for lane in range(1, 30):
            self.add_lane(lane)

    def add_lane(self, lane):

        self.lanes.append(RoadLane(lane))

        if lane % 2 == 0:

            direction = 1 if lane % 4 == 0 else -1

            count = min(2 + self.level // 2, 5)

            for i in range(count):

                if random.random() < 0.7:
                    vehicle = Car(lane, direction, self.level)
                else:
                    vehicle = Truck(lane, direction, self.level)

                vehicle.x = random.randint(-100, WIDTH)
                self.vehicles.append(vehicle)

            if random.random() < 0.4:
                self.coins.append(Coin(lane))

    def add_score(self, value):

        self.score += value
        self.player.score = self.score

        if self.score > self.record:
            self.record = self.score
            save_record(self.record)

        new_level = self.score // 5 + 1

        if new_level > self.level:
            self.level = new_level

            for vehicle in self.vehicles:
                vehicle.speed += 15

    def check_coin(self):

        for coin in self.coins:

            if (
                not coin.isCollected
                and self.player.x == coin.x
                and self.player.y == coin.lane
            ):

                coin.isCollected = True
                self.add_score(1)

    def check_collision(self):

        player_rect = pygame.Rect(
            self.player.x * CELL + 12,
            HEIGHT - 120 -
            (self.player.y - self.camera) * CELL + 12,
            36, 36
        )

        for vehicle in self.vehicles:

            if vehicle.lane == self.player.y:

                if player_rect.colliderect(
                    vehicle.get_rect(self.camera)
                ):

                    self.endGame()
                    return

    def endGame(self):

        self.gameOver = True
        self.player.isAlive = False

    def update(self, dt):

        if self.gameOver:
            return

        for vehicle in self.vehicles:
            vehicle.update(dt)

        self.check_collision()
        self.check_coin()

        if self.player.y > self.camera + 4:
            self.camera = self.player.y - 4

        highest = max(lane.number for lane in self.lanes)

        while highest < self.player.y + 20:
            highest += 1
            self.add_lane(highest)

        self.vehicles = [
            v for v in self.vehicles
            if v.lane > self.camera - 3
        ]

        self.coins = [
            c for c in self.coins
            if c.lane > self.camera - 3
        ]

    def draw(self):

        screen.fill(GREEN)

        for lane in self.lanes:
            lane.draw(self.camera)

        for coin in self.coins:
            coin.draw(self.camera)

        for vehicle in self.vehicles:
            vehicle.draw(self.camera)

        if self.player.isAlive:
            self.player.draw(self.camera)

        # Ақпарат панелі
        pygame.draw.rect(
            screen, BLACK,
            (0, 0, WIDTH, 55)
        )

        texts = [
            f"Ұпай: {self.score}",
            f"Рекорд: {self.record}",
            f"Деңгей: {self.level}"
        ]

        for i, item in enumerate(texts):

            txt = small_font.render(item, True, WHITE)

            screen.blit(txt, (10 + i * 200, 17))

        if self.gameOver:
            self.draw_game_over()

    def draw_game_over(self):

        overlay = pygame.Surface((WIDTH, HEIGHT))
        overlay.set_alpha(200)
        overlay.fill(BLACK)

        screen.blit(overlay, (0, 0))

        title = big_font.render(
            "GAME OVER", True, RED
        )

        score_text = font.render(
            f"Ұпай: {self.score}", True, WHITE
        )

        restart = font.render(
            "R - Қайта бастау", True, YELLOW
        )

        screen.blit(
            title,
            (WIDTH // 2 - title.get_width() // 2, 270)
        )

        screen.blit(
            score_text,
            (WIDTH // 2 - score_text.get_width() // 2, 340)
        )

        screen.blit(
            restart,
            (WIDTH // 2 - restart.get_width() // 2, 400)
        )


# =====================================
# MAIN PROGRAM
# =====================================

def main():

    game = GameManager()

    running = True

    while running:

        dt = min(clock.tick(FPS) / 1000, 0.05)

        for event in pygame.event.get():

            if event.type == pygame.QUIT:
                running = False

            if event.type == pygame.KEYDOWN:

                if event.key == pygame.K_ESCAPE:
                    running = False

                if game.gameOver:

                    if event.key == pygame.K_r:
                        game.startGame()

                else:

                    if event.key in (pygame.K_UP, pygame.K_w):
                        game.player.move("up")

                    elif event.key in (pygame.K_DOWN, pygame.K_s):
                        game.player.move("down")

                    elif event.key in (pygame.K_LEFT, pygame.K_a):
                        game.player.move("left")

                    elif event.key in (pygame.K_RIGHT, pygame.K_d):
                        game.player.move("right")

                    if (
                        game.player.y >= game.last_safe_lane + 5
                    ):

                        game.add_score(1)
                        game.last_safe_lane += 5

                    game.check_coin()
                    game.check_collision()

        game.update(dt)
        game.draw()

        pygame.display.flip()

    pygame.quit()
    sys.exit()


if __name__ == "__main__":
    main()