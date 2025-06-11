import pygame
import sys
import random
import math
import time
from collections import deque
import tkinter as tk
from tkinter import filedialog

SEEKER_SIZE = (20, 20)
PACKAGE_SIZE = (18, 18)
HIDE_SIZE = (20, 20)

WALKABLE_MIN = (90, 90, 90)
WALKABLE_MAX = (150, 150, 150)

pygame.init()
font = pygame.font.SysFont(None, 36)
button_font = pygame.font.SysFont(None, 28)
clock = pygame.time.Clock()

def load_image(path, size):
    img = pygame.image.load(path).convert_alpha()
    return pygame.transform.smoothscale(img, size)

def select_map_file_dialog():
    root = tk.Tk()
    root.withdraw()
    file_path = filedialog.askopenfilename(title="Pilih file map", filetypes=[("Image Files", "*.png;*.jpg;*.jpeg")])
    root.destroy()
    return file_path

def is_road(color):
    return all(WALKABLE_MIN[i] <= color[i] <= WALKABLE_MAX[i] for i in range(3))

def get_random_road_position():
    for _ in range(1000):
        x = random.randint(0, WIDTH - 1)
        y = random.randint(0, HEIGHT - 1)
        if is_road(map_image.get_at((x, y))[:3]):
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
        for dx, dy in [(-1,0), (1,0), (0,-1), (0,1), (-1,-1), (1,1), (-1,1), (1,-1)]:
            nx, ny = x + dx, y + dy
            if 0 <= nx < WIDTH and 0 <= ny < HEIGHT:
                if (nx, ny) not in visited:
                    if is_road(map_image.get_at((nx, ny))[:3]):
                        # Validasi agar tidak menabrak sudut saat diagonal
                        if abs(dx) + abs(dy) == 2:
                            if not (is_road(map_image.get_at((x + dx, y))[:3]) and is_road(map_image.get_at((x, y + dy))[:3])):
                                continue
                        visited.add((nx, ny))
                        queue.append(((nx, ny), path + [(nx, ny)]))
    return []

def reset_game():
    global path_index, start_time, end_time, elapsed_time
    global seeker_pos, path, pickup_done, paused, game_active, stage
    path_index = 0
    start_time = time.time()
    end_time = None
    elapsed_time = 0
    seeker_pos = start
    path = find_path(start, package_pos)
    pickup_done = False
    paused = False
    game_active = True
    stage = "pickup"

def show_end_screen():
    screen.fill((240, 240, 240))
    title = font.render("Paket telah diantar!", True, (0, 150, 0))
    screen.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT // 2 - 80))
    time_display = font.render(f"Waktu: {elapsed_time} detik", True, (0, 0, 0))
    screen.blit(time_display, (WIDTH // 2 - time_display.get_width() // 2, HEIGHT // 2 - 30))

    again_button = pygame.Rect(WIDTH // 2 - 160, HEIGHT // 2 + 30, 140, 40)
    exit_button = pygame.Rect(WIDTH // 2 + 20, HEIGHT // 2 + 30, 140, 40)
    draw_button(again_button, "Main Lagi")
    draw_button(exit_button, "Keluar")
    pygame.display.flip()

    waiting = True
    while waiting:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            elif event.type == pygame.MOUSEBUTTONDOWN:
                if again_button.collidepoint(event.pos):
                    reset_game()
                    return True
                elif exit_button.collidepoint(event.pos):
                    pygame.quit()
                    sys.exit()
        clock.tick(60)

def draw_button(rect, label):
    mouse_pos = pygame.mouse.get_pos()
    hover = rect.collidepoint(mouse_pos)
    color = (70, 130, 180) if not hover else (100, 160, 210)
    pygame.draw.rect(screen, color, rect, border_radius=8)
    text = button_font.render(label, True, (255, 255, 255))
    text_rect = text.get_rect(center=rect.center)
    screen.blit(text, text_rect)

screen = pygame.display.set_mode((1200, 800))
pygame.display.set_caption("Smart Kurir")

map_loaded = False
map_image = None
WIDTH = HEIGHT = 0
start = package_pos = hide_pos = seeker_pos = None
path = []
path_index = 0
start_time = end_time = None
elapsed_time = 0
pickup_done = False
paused = False
game_active = False
stage = "pickup"

seek_icon = load_image("seek_arrow.png", SEEKER_SIZE)
hide_icon = load_image("hide_shield.png", HIDE_SIZE)
package_icon = load_image("paket.jpg", PACKAGE_SIZE)

running = True
while running:
    screen.fill((230, 230, 230))
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        elif event.type == pygame.MOUSEBUTTONDOWN:
            if map_loaded:
                if start_button.collidepoint(event.pos):
                    reset_game()
                elif shuffle_button.collidepoint(event.pos):
                    start = get_random_road_position()
                    package_pos = get_random_road_position()
                    hide_pos = get_random_road_position()
                    seeker_pos = start
                    path = find_path(start, package_pos)
                    path_index = 0
                    pickup_done = False
                    paused = False
                    game_active = False
                elif pause_button.collidepoint(event.pos):
                    if game_active:
                        paused = not paused
            else:
                if shuffle_button.collidepoint(event.pos):
                    file = select_map_file_dialog()
                    if file:
                        img = pygame.image.load(file)
                        w, h = img.get_size()
                        if 1000 <= w <= 1500 and 700 <= h <= 1000:
                            map_image = img
                            WIDTH, HEIGHT = map_image.get_size()
                            screen = pygame.display.set_mode((WIDTH, HEIGHT))
                            start = get_random_road_position()
                            package_pos = get_random_road_position()
                            hide_pos = get_random_road_position()
                            seeker_pos = start
                            path = find_path(start, package_pos)
                            map_loaded = True

    if map_loaded:
        screen.blit(map_image, (0, 0))

        start_button = pygame.Rect(WIDTH - 280, 20, 80, 40)
        shuffle_button = pygame.Rect(WIDTH - 190, 20, 80, 40)
        pause_button = pygame.Rect(WIDTH - 100, 20, 80, 40)

        draw_button(start_button, "START")
        draw_button(shuffle_button, "ACAK")
        draw_button(pause_button, "JEDA" if not paused else "LANJUT")

        if start_time and not end_time:
            elapsed_time = int(time.time() - start_time)
        time_display = font.render(f"Waktu: {elapsed_time} detik", True, (0, 0, 0))
        screen.blit(time_display, (20, 20))

        if seeker_pos:
            if path_index < len(path):
                seeker_pos = path[path_index]
                if path_index + 1 < len(path):
                    dx = path[path_index + 1][0] - seeker_pos[0]
                    dy = path[path_index + 1][1] - seeker_pos[1]
                    angle = math.degrees(math.atan2(-dy, dx))
                else:
                    angle = 0
                rotated_seek = pygame.transform.rotate(seek_icon, angle)
                screen.blit(rotated_seek, rotated_seek.get_rect(center=seeker_pos))
            else:
                screen.blit(seek_icon, seek_icon.get_rect(center=seeker_pos))

            if not pickup_done:
                screen.blit(package_icon, package_icon.get_rect(center=package_pos))

            screen.blit(hide_icon, hide_icon.get_rect(center=hide_pos))

        if game_active and not paused and path_index < len(path):
            path_index += 1
            if stage == "pickup" and seeker_pos == package_pos:
                pickup_done = True
                stage = "deliver"
                path = find_path(seeker_pos, hide_pos)
                path_index = 0
            elif stage == "deliver" and seeker_pos == hide_pos:
                game_active = False
                end_time = time.time()
                elapsed_time = int(end_time - start_time)
                if show_end_screen():
                    continue
    else:
        title = font.render("SMART KURIR", True, (0, 100, 200))
        screen.blit(title, (screen.get_width() // 2 - title.get_width() // 2, 250))
        shuffle_button = pygame.Rect(screen.get_width() // 2 - 40, 350, 80, 40)
        draw_button(shuffle_button, "PILIH PETA")

    pygame.display.flip()
    clock.tick(60)

pygame.quit()
sys.exit()
