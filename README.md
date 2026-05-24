import pygame
import random
import sys
import os
import time
import json

# Инициализация
pygame.init()

# Размеры экрана
WIDTH, HEIGHT = 800, 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Golden Catcher")

# Цвета (базовые)
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
GOLD = (255, 215, 0)
ORANGE = (255, 140, 0)
RED = (255, 0, 0)
GRAY = (100, 100, 100)
LIGHT_GRAY = (200, 200, 200)
SKY_BLUE = (135, 206, 235)
GRASS_GREEN = (34, 139, 34)
YELLOW = (255, 255, 0)
GREEN = (0, 200, 0)
PURPLE = (128, 0, 128)
DARK_BLUE = (0, 0, 139)
BLUE = (30, 144, 255)

# Шрифты
font_title = pygame.font.Font(None, 64)
font_menu = pygame.font.Font(None, 42)
font_med = pygame.font.Font(None, 36)
font_small = pygame.font.Font(None, 28)

# ------------------ СКИНЫ ------------------
SKINS = {
    "Standart": {"price": 0, "color": BLUE, "name_ru": "Стандартный"},
    "Gold":     {"price": 100, "color": GOLD, "name_ru": "Золотой"},
    "Red":      {"price": 80, "color": RED, "name_ru": "Красный"},
    "Green":    {"price": 80, "color": GREEN, "name_ru": "Зелёный"},
    "Purple":   {"price": 120, "color": PURPLE, "name_ru": "Фиолетовый"},
}
skin_names = list(SKINS.keys())

# Загрузка данных о скинах
def load_skin_data():
    default = {"owned": ["Standart"], "current": "Standart"}
    if os.path.exists("skins.json"):
        try:
            with open("skins.json", "r") as f:
                data = json.load(f)
                return data
        except:
            return default
    return default

def save_skin_data(data):
    with open("skins.json", "w") as f:
        json.dump(data, f)

skin_data = load_skin_data()
current_skin = skin_data["current"]
owned_skins = set(skin_data["owned"])

# ------------------ ВАЛЮТА ------------------
def load_coins():
    if os.path.exists("coins.txt"):
        try:
            with open("coins.txt", "r") as f:
                return int(f.read())
        except:
            return 0
    return 0

def save_coins(coins):
    with open("coins.txt", "w") as f:
        f.write(str(coins))

total_coins = load_coins()  # общая валюта, заработанная за все игры

# Параметры сложности
DIFFICULTY = {
    "Easy":     {"cart_speed": 12, "coin_min": 3, "coin_max": 6,  "spawn_delay": 35},
    "Normal":   {"cart_speed": 18, "coin_min": 4, "coin_max": 9,  "spawn_delay": 28},
    "Hard":     {"cart_speed": 24, "coin_min": 6, "coin_max": 12, "spawn_delay": 22},
    "Extreme":  {"cart_speed": 30, "coin_min": 8, "coin_max": 16, "spawn_delay": 18},
}
difficulty_names = ["Easy", "Normal", "Hard", "Extreme"]
current_difficulty = 1
TIME_OPTIONS = [60, 120, 180, 300]
time_index = 2

# Глобальные переменные игры
game_active = False
score = 0
cart_x = WIDTH//2 - 60
coins_list = []
start_time = 0
game_duration = TIME_OPTIONS[time_index]
final_score = 0
highscore = 0
cart_speed = DIFFICULTY[difficulty_names[current_difficulty]]["cart_speed"]

# Загрузка рекорда
if os.path.exists("highscore.txt"):
    with open("highscore.txt", "r") as f:
        try:
            highscore = int(f.read())
        except:
            highscore = 0

# Сенсорные зоны для управления
LEFT_ZONE = pygame.Rect(0, HEIGHT - 150, WIDTH//2, 150)
RIGHT_ZONE = pygame.Rect(WIDTH//2, HEIGHT - 150, WIDTH//2, 150)

left_held = False
right_held = False
left_pressed = False
right_pressed = False

MENU_BUTTON_RECT = pygame.Rect(WIDTH - 100, 10, 90, 40)

# Звук
try:
    catch_sound = pygame.mixer.Sound(pygame.mixer.Sound(buffer=bytes([0x52,0x49,0x46,0x46,0x24,0x00,0x00,0x00,0x57,0x41,0x56,0x45,0x66,0x6d,0x74,0x20,0x10,0x00,0x00,0x00,0x01,0x00,0x01,0x00,0x44,0xac,0x00,0x00,0x44,0xac,0x00,0x00,0x01,0x00,0x08,0x00,0x64,0x61,0x74,0x61,0x02,0x00,0x00,0x00,0x80,0x80])))
except:
    catch_sound = None

clock = pygame.time.Clock()

# --- Функции рисования с учётом скина ---
def draw_cart_with_skin(x, y, pressed=False):
    skin_color = SKINS[current_skin]["color"]
    pygame.draw.rect(screen, BLACK, (x+3, y+3, 120, 30))
    color = (skin_color[0]//2, skin_color[1]//2, skin_color[2]//2) if pressed else skin_color
    pygame.draw.rect(screen, color, (x, y, 120, 30), border_radius=8)
    pygame.draw.rect(screen, WHITE, (x, y, 120, 30), 2, border_radius=8)
    # Декоративная полоска всегда золотая
    pygame.draw.line(screen, GOLD, (x+10, y+5), (x+110, y+5), 3)
    pygame.draw.circle(screen, BLACK, (x+15, y+25), 8)
    pygame.draw.circle(screen, BLACK, (x+105, y+25), 8)
    pygame.draw.circle(screen, GRAY, (x+15, y+25), 5)
    pygame.draw.circle(screen, GRAY, (x+105, y+25), 5)

def draw_gradient_background():
    for y in range(HEIGHT):
        if y < HEIGHT // 2:
            t = y / (HEIGHT//2)
            color = (int(SKY_BLUE[0] * (1-t) + GRASS_GREEN[0]*t),
                     int(SKY_BLUE[1] * (1-t) + GRASS_GREEN[1]*t),
                     int(SKY_BLUE[2] * (1-t) + GRASS_GREEN[2]*t))
        else:
            t = (y - HEIGHT//2) / (HEIGHT//2)
            color = (int(GRASS_GREEN[0] * (1-t) + DARK_BLUE[0]*t),
                     int(GRASS_GREEN[1] * (1-t) + DARK_BLUE[1]*t),
                     int(GRASS_GREEN[2] * (1-t) + DARK_BLUE[2]*t))
        pygame.draw.line(screen, color, (0, y), (WIDTH, y))

def draw_coin(x, y, size):
    pygame.draw.circle(screen, GOLD, (x+size//2, y+size//2), size//2)
    pygame.draw.circle(screen, ORANGE, (x+size//2, y+size//2), size//2, 2)
    pygame.draw.circle(screen, WHITE, (x+size//3, y+size//3), size//6)

def draw_joystick(rect, direction, pressed=False):
    s = pygame.Surface((rect.width, rect.height), pygame.SRCALPHA)
    alpha = 100 if not pressed else 180
    s.fill((255,255,255,alpha))
    screen.blit(s, (rect.x, rect.y))
    pygame.draw.rect(screen, WHITE, rect, 3, border_radius=20)
    font = pygame.font.Font(None, 60)
    arrow = font.render(direction, True, WHITE)
    arrow_rect = arrow.get_rect(center=rect.center)
    screen.blit(arrow, arrow_rect)

# --- Экран магазина ---
def shop_screen():
    global total_coins, current_skin, owned_skins, skin_data
    running_shop = True
    scroll = 0  # для прокрутки, если будет много скинов
    while running_shop:
        screen.fill((20,20,40))
        title = font_title.render("MAGAZIN", True, GOLD)
        screen.blit(title, (WIDTH//2 - title.get_width()//2, 30))
        
        # Отображение баланса
        coin_text = font_med.render(f"💎 {total_coins}", True, YELLOW)
        screen.blit(coin_text, (WIDTH - 150, 20))
        
        # Список скинов
        y_offset = 120
        for i, skin_id in enumerate(skin_names):
            skin = SKINS[skin_id]
            color_preview = skin["color"]
            price = skin["price"]
            name = skin["name_ru"]
            owned = skin_id in owned_skins
            selected = (skin_id == current_skin)
            
            # Фон для каждого скина
            bg_rect = pygame.Rect(50, y_offset + i*70, WIDTH-100, 60)
            if selected:
                pygame.draw.rect(screen, DARK_BLUE, bg_rect, border_radius=10)
            else:
                pygame.draw.rect(screen, (50,50,80), bg_rect, border_radius=10)
            pygame.draw.rect(screen, WHITE, bg_rect, 2, border_radius=10)
            
            # Превью цвета
            preview_rect = pygame.Rect(70, y_offset + i*70 + 10, 40, 40)
            pygame.draw.rect(screen, color_preview, preview_rect, border_radius=5)
            pygame.draw.rect(screen, WHITE, preview_rect, 1, border_radius=5)
            
            # Название
            name_surf = font_med.render(name, True, WHITE)
            screen.blit(name_surf, (130, y_offset + i*70 + 15))
            
            # Цена или статус
            if owned:
                status = "✅ Куплен" if not selected else "▶ Выбран"
                status_color = GREEN if not selected else YELLOW
                status_surf = font_small.render(status, True, status_color)
                screen.blit(status_surf, (WIDTH - 180, y_offset + i*70 + 20))
            else:
                price_surf = font_med.render(f"{price} 💎", True, GOLD)
                screen.blit(price_surf, (WIDTH - 180, y_offset + i*70 + 15))
            
            # Кнопка "Купить/Выбрать"
            btn_rect = pygame.Rect(WIDTH - 100, y_offset + i*70 + 15, 70, 30)
            if owned:
                btn_color = GREEN if not selected else DARK_BLUE
                btn_text = "Выбрать" if not selected else "Выбран"
            else:
                btn_color = GRAY
                btn_text = "Купить"
            pygame.draw.rect(screen, btn_color, btn_rect, border_radius=5)
            pygame.draw.rect(screen, WHITE, btn_rect, 1, border_radius=5)
            btn_label = font_small.render(btn_text, True, WHITE)
            screen.blit(btn_label, (btn_rect.x + (70 - btn_label.get_width())//2, btn_rect.y + 5))
            setattr(shop_screen, f"btn_{i}", (btn_rect, skin_id))
        
        # Кнопка "Назад"
        back_rect = pygame.Rect(WIDTH//2 - 100, HEIGHT - 80, 200, 50)
        pygame.draw.rect(screen, RED, back_rect, border_radius=10)
        pygame.draw.rect(screen, WHITE, back_rect, 3, border_radius=10)
        back_text = font_menu.render("Назад", True, WHITE)
        screen.blit(back_text, (back_rect.x + (200 - back_text.get_width())//2, back_rect.y + 8))
        
        pygame.display.flip()
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                x, y = event.pos
                if back_rect.collidepoint(x, y):
                    running_shop = False
                # Проверка кнопок скинов
                for i in range(len(skin_names)):
                    if hasattr(shop_screen, f"btn_{i}"):
                        btn_rect, skin_id = getattr(shop_screen, f"btn_{i}")
                        if btn_rect.collidepoint(x, y):
                            skin = SKINS[skin_id]
                            if skin_id in owned_skins:
                                # Просто выбрать
                                current_skin = skin_id
                                skin_data["current"] = current_skin
                                save_skin_data(skin_data)
                            else:
                                # Купить, если хватает монет
                                if total_coins >= skin["price"]:
                                    total_coins -= skin["price"]
                                    owned_skins.add(skin_id)
                                    current_skin = skin_id
                                    skin_data["owned"] = list(owned_skins)
                                    skin_data["current"] = current_skin
                                    save_skin_data(skin_data)
                                    save_coins(total_coins)
                                else:
                                    # звук ошибки? просто игнор
                                    pass
        clock.tick(30)

def draw_menu():
    global current_difficulty, time_index, total_coins
    screen.fill((20, 20, 40))
    title = font_title.render("GOLDEN CATCHER", True, GOLD)
    screen.blit(title, (WIDTH//2 - title.get_width()//2, 50))
    
    # Баланс монет
    coin_text = font_med.render(f"💎 {total_coins}", True, YELLOW)
    screen.blit(coin_text, (WIDTH - 150, 30))
    
    # Кнопка магазина
    shop_btn = pygame.Rect(WIDTH - 180, 30, 100, 40)
    pygame.draw.rect(screen, PURPLE, shop_btn, border_radius=10)
    pygame.draw.rect(screen, WHITE, shop_btn, 2, border_radius=10)
    shop_label = font_small.render("Магазин", True, WHITE)
    screen.blit(shop_label, (shop_btn.x + (100 - shop_label.get_width())//2, shop_btn.y + 10))
    draw_menu.shop_btn = shop_btn
    
    # Сложность
    diff_text = font_menu.render("Difficulty:", True, WHITE)
    screen.blit(diff_text, (WIDTH//2 - 150, 160))
    for i, name in enumerate(difficulty_names):
        color = YELLOW if i == current_difficulty else WHITE
        btn_rect = pygame.Rect(WIDTH//2 - 80 + i*110, 200, 100, 40)
        pygame.draw.rect(screen, GRAY if i!=current_difficulty else DARK_BLUE, btn_rect, border_radius=10)
        pygame.draw.rect(screen, WHITE, btn_rect, 2, border_radius=10)
        text = font_menu.render(name, True, color)
        screen.blit(text, (btn_rect.x + (100 - text.get_width())//2, btn_rect.y + 5))
        setattr(draw_menu, f"diff_btn_{i}", btn_rect)
    
    # Время
    time_text = font_menu.render("Time:", True, WHITE)
    screen.blit(time_text, (WIDTH//2 - 150, 280))
    time_labels = ["1 min", "2 min", "3 min", "5 min"]
    for i, label in enumerate(time_labels):
        color = YELLOW if i == time_index else WHITE
        btn_rect = pygame.Rect(WIDTH//2 - 80 + i*110, 320, 100, 40)
        pygame.draw.rect(screen, GRAY if i!=time_index else DARK_BLUE, btn_rect, border_radius=10)
        pygame.draw.rect(screen, WHITE, btn_rect, 2, border_radius=10)
        text = font_menu.render(label, True, color)
        screen.blit(text, (btn_rect.x + (100 - text.get_width())//2, btn_rect.y + 5))
        setattr(draw_menu, f"time_btn_{i}", btn_rect)
    
    # Старт
    start_rect = pygame.Rect(WIDTH//2 - 100, 400, 200, 50)
    pygame.draw.rect(screen, GREEN, start_rect, border_radius=15)
    pygame.draw.rect(screen, WHITE, start_rect, 3, border_radius=15)
    start_txt = font_menu.render("START GAME", True, WHITE)
    screen.blit(start_txt, (start_rect.x + (200 - start_txt.get_width())//2, start_rect.y + 8))
    draw_menu.start_rect = start_rect
    
    # Рекорд
    high_txt = font_small.render(f"BEST SCORE: {highscore}", True, GOLD)
    screen.blit(high_txt, (WIDTH//2 - high_txt.get_width()//2, HEIGHT - 50))

def reset_game():
    global score, coins_list, cart_x, start_time, game_active, final_score, left_held, right_held
    global cart_speed, game_duration
    score = 0
    coins_list.clear()
    cart_x = WIDTH//2 - 60
    game_active = True
    final_score = 0
    left_held = False
    right_held = False
    left_pressed = False
    right_pressed = False
    start_time = time.time()
    diff = difficulty_names[current_difficulty]
    cart_speed = DIFFICULTY[diff]["cart_speed"]
    game_duration = TIME_OPTIONS[time_index]

def game_loop():
    global score, coins_list, cart_x, start_time, game_active, final_score, left_held, right_held, left_pressed, right_pressed
    global highscore, game_duration, total_coins
    reset_game()
    
    diff = difficulty_names[current_difficulty]
    coin_min_speed = DIFFICULTY[diff]["coin_min"]
    coin_max_speed = DIFFICULTY[diff]["coin_max"]
    coin_spawn_delay = DIFFICULTY[diff]["spawn_delay"]
    
    coin_spawn_timer = 0
    running_game = True
    
    while running_game:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                x, y = event.pos
                if MENU_BUTTON_RECT.collidepoint(x, y):
                    game_active = False
                    running_game = False
                    break
                if game_active:
                    if LEFT_ZONE.collidepoint(x, y):
                        left_held = True
                        left_pressed = True
                    if RIGHT_ZONE.collidepoint(x, y):
                        right_held = True
                        right_pressed = True
            if event.type == pygame.MOUSEBUTTONUP:
                left_held = False
                right_held = False
                left_pressed = False
                right_pressed = False
        
        if not running_game:
            break
        
        if game_active:
            if left_held and not right_held:
                cart_x -= cart_speed
            elif right_held and not left_held:
                cart_x += cart_speed
            cart_x = max(0, min(WIDTH - 120, cart_x))
            
            elapsed = time.time() - start_time
            remaining = max(0, game_duration - elapsed)
            if remaining <= 0:
                game_active = False
                final_score = score
                # Добавляем заработанные монеты к общей валюте
                total_coins += score
                save_coins(total_coins)
                if score > highscore:
                    highscore = score
                    with open("highscore.txt", "w") as f:
                        f.write(str(highscore))
                show_game_over_screen()
                running_game = False
                break
            
            coin_spawn_timer += 1
            if coin_spawn_timer >= coin_spawn_delay:
                coin_spawn_timer = 0
                coin_size = random.randint(20, 35)
                coin_x = random.randint(0, WIDTH - coin_size)
                coin_y = -coin_size
                speed = random.uniform(coin_min_speed, coin_max_speed)
                coins_list.append([coin_x, coin_y, coin_size, speed])
            
            new_coins = []
            for coin in coins_list:
                coin[1] += coin[3]
                if (cart_x < coin[0] + coin[2] and cart_x + 120 > coin[0] and
                    570 < coin[1] + coin[2] and 570 + 30 > coin[1]):
                    score += 1
                    if catch_sound:
                        catch_sound.play()
                    continue
                if coin[1] > HEIGHT:
                    continue
                new_coins.append(coin)
            coins_list = new_coins
        
        draw_gradient_background()
        for coin in coins_list:
            draw_coin(coin[0], coin[1], coin[2])
        draw_cart_with_skin(cart_x, 570, left_pressed or right_pressed)
        draw_joystick(LEFT_ZONE, "◀", left_pressed)
        draw_joystick(RIGHT_ZONE, "▶", right_pressed)
        
        score_text = font_med.render(f"Score: {score}", True, WHITE)
        screen.blit(score_text, (10, 10))
        minutes = int(remaining) // 60
        seconds = int(remaining) % 60
        time_text = font_med.render(f"Time: {minutes:02d}:{seconds:02d}", True, YELLOW)
        screen.blit(time_text, (WIDTH - time_text.get_width() - 10, 10))
        
        pygame.draw.rect(screen, RED, MENU_BUTTON_RECT, border_radius=10)
        pygame.draw.rect(screen, WHITE, MENU_BUTTON_RECT, 2, border_radius=10)
        menu_txt = font_small.render("MENU", True, WHITE)
        screen.blit(menu_txt, (MENU_BUTTON_RECT.x + (90 - menu_txt.get_width())//2, MENU_BUTTON_RECT.y + 8))
        
        pygame.display.flip()
        clock.tick(60)

def show_game_over_screen():
    screen.fill(BLACK)
    overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
    overlay.fill((0,0,0,200))
    screen.blit(overlay, (0,0))
    over_text = font_title.render("TIME'S UP!", True, GOLD)
    score_text = font_med.render(f"Your score: {final_score}", True, WHITE)
    high_text = font_med.render(f"Best: {highscore}", True, LIGHT_GRAY)
    coin_earned = font_med.render(f"+{final_score} 💎", True, YELLOW)
    restart_text = font_small.render("Tap anywhere to continue", True, WHITE)
    screen.blit(over_text, (WIDTH//2 - over_text.get_width()//2, HEIGHT//3))
    screen.blit(score_text, (WIDTH//2 - score_text.get_width()//2, HEIGHT//2))
    screen.blit(high_text, (WIDTH//2 - high_text.get_width()//2, HEIGHT//2 + 40))
    screen.blit(coin_earned, (WIDTH//2 - coin_earned.get_width()//2, HEIGHT//2 + 80))
    screen.blit(restart_text, (WIDTH//2 - restart_text.get_width()//2, HEIGHT//2 + 130))
    pygame.display.flip()
    waiting = True
    while waiting:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                waiting = False
        clock.tick(30)

def main():
    global current_difficulty, time_index, total_coins
    running = True
    while running:
        draw_menu()
        pygame.display.flip()
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN:
                x, y = event.pos
                # Магазин
                if hasattr(draw_menu, "shop_btn") and draw_menu.shop_btn.collidepoint(x, y):
                    shop_screen()
                    # после магазина обновляем баланс на экране меню
                    total_coins = load_coins()
                    continue
                # Кнопки сложности
                for i in range(4):
                    btn = getattr(draw_menu, f"diff_btn_{i}", None)
                    if btn and btn.collidepoint(x, y):
                        current_difficulty = i
                # Кнопки времени
                for i in range(4):
                    btn = getattr(draw_menu, f"time_btn_{i}", None)
                    if btn and btn.collidepoint(x, y):
                        time_index = i
                # Старт
                if draw_menu.start_rect.collidepoint(x, y):
                    game_loop()
                    total_coins = load_coins()  # обновить валюту после игры
        clock.tick(30)

if __name__ == "__main__":
    main()# my-game-1