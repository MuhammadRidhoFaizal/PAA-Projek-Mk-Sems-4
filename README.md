import pygame
import sys
import random
import math
import time
from collections import deque

MAP_PATH = "MAP.png"
SEEK_IMAGE_PATH = "seek_arrow.png"
HIDE_IMAGE_PATH = "hide_shield.png"
WALKABLE_COLOR = (150, 150, 150)
ICON_SIZE = (40, 40)

pygame.init()
map_image = pygame.image.load(MAP_PATH)
WIDTH, HEIGHT = map_image.get_size()
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Hide and Seek")

font = pygame.font.SysFont(None, 36)
big_font = pygame.font.SysFont(None, 64)
clock = pygame.time.Clock()

# Load dan resize ikon
def load_image(path):
    img = pygame.image.load(path).convert_alpha()
    return pygame.transform.smoothscale(img, ICON_SIZE)

seek_icon = load_image(SEEK_IMAGE_PATH)
hide_icon = load_image(HIDE_IMAGE_PATH)

def get_random_edge_position():
    for _ in range(1000):
        side = random.choice(["top", "bottom", "left", "right"])
        if side == "top":
            x, y = random.randint(0, WIDTH - 1), 5
        elif side == "bottom":
            x, y = random.randint(0, WIDTH - 1), HEIGHT - 6
        elif side == "left":
            x, y = 5, random.randint(0, HEIGHT - 1)
        else:
            x, y = WIDTH - 6, random.randint(0, HEIGHT - 1)

        if map_image.get_at((x, y))[:3] == WALKABLE_COLOR:
            return x, y
    return None

def find_path(start, goal):
    queue = deque()
    queue.append((start, [start]))
    visited = set()
    visited.add(start)

    while queue:
        (x, y), path = queue.popleft()

        if (x, y) == goal:
            return path

        for dx, dy in [(-1,0), (1,0), (0,-1), (0,1)]:
            nx, ny = x + dx, y + dy
            if 0 <= nx < WIDTH and 0 <= ny < HEIGHT:
                if (nx, ny) not in visited:
                    if map_image.get_at((nx, ny))[:3] == WALKABLE_COLOR:
                        visited.add((nx, ny))
                        queue.append(((nx, ny), path + [(nx, ny)]))
    return []

def rotate_image(image, angle):
    return pygame.transform.rotate(image, -angle)

def show_centered_text(text, font, color, duration=3):
    start_time = time.time()
    while time.time() - start_time < duration:
        screen.blit(map_image, (0, 0))
        text_surf = font.render(text, True, color)
        text_rect = text_surf.get_rect(center=(WIDTH // 2, HEIGHT // 2))
        screen.blit(text_surf, text_rect)
        pygame.display.flip()
        clock.tick(60)

start_button = pygame.Rect(10, 10, 80, 40)
stop_button = pygame.Rect(100, 10, 80, 40)

running = True
game_active = False
path = []
start_time = None
elapsed_time = 0
path_index = 0
show_found_timer = 0

show_centered_text("Halo, semoga menyenangkan!", big_font, (0, 0, 0), duration=2)

while running:
    screen.blit(map_image, (0, 0))
    mouse_pos = pygame.mouse.get_pos()

    # Tombol
    pygame.draw.rect(screen, (0, 200, 0), start_button)
    pygame.draw.rect(screen, (200, 0, 0), stop_button)
    screen.blit(font.render("START", True, (255,255,255)), (15, 15))
    screen.blit(font.render("STOP", True, (255,255,255)), (110, 15))

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

        elif event.type == pygame.MOUSEBUTTONDOWN:
            if start_button.collidepoint(mouse_pos):
                start = get_random_edge_position()
                end = get_random_edge_position()
                if start and end:
                    path = find_path(start, end)
                    path_index = 0
                    start_time = time.time()
                    elapsed_time = 0
                    game_active = True
                    hide_pos = end
                    show_found_timer = 0
                    print("Start:", start, "End:", end)
                else:
                    print("Tidak ditemukan titik jalan valid di tepi.")
            elif stop_button.collidepoint(mouse_pos):
                game_active = False

    # Game aktif
    if game_active and path and path_index < len(path):
        seek_pos = path[path_index]

        if path_index + 1 < len(path):
            next_pos = path[path_index + 1]
            dx = next_pos[0] - seek_pos[0]
            dy = next_pos[1] - seek_pos[1]
            angle = math.degrees(math.atan2(dy, dx))  # <- arah benar
        else:
            angle = 0

        rotated_seek = rotate_image(seek_icon, angle)

        # Gambar seeker dan hider
        seek_rect = rotated_seek.get_rect(center=seek_pos)
        screen.blit(rotated_seek, seek_rect)

        hide_rect = hide_icon.get_rect(center=hide_pos)
        screen.blit(hide_icon, hide_rect)

        path_index += 1
        elapsed_time = int(time.time() - start_time)

        # Cek apakah seeker menyentuh hider
        if abs(seek_pos[0] - hide_pos[0]) < 3 and abs(seek_pos[1] - hide_pos[1]) < 3:
            show_found_timer = time.time()
            game_active = False

    elif show_found_timer > 0 and time.time() - show_found_timer < 5:
        text = big_font.render("Target ditemukan", True, (255, 0, 0))
        rect = text.get_rect(center=(WIDTH // 2, HEIGHT // 2))
        screen.blit(text, rect)

    elif show_found_timer > 0 and time.time() - show_found_timer >= 5:
        show_centered_text("Terima kasih telah bermain!", big_font, (0, 0, 0), duration=3)
        show_found_timer = 0

    # Timer tampil
    if start_time and not show_found_timer:
        time_text = font.render(f"Waktu: {elapsed_time} detik", True, (0, 0, 0))
        screen.blit(time_text, (WIDTH // 2 - 80, 10))

    pygame.display.flip()
    clock.tick(60)

pygame.quit()
sys.exit()
