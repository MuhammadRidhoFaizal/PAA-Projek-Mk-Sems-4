# PAA-Projek-Mk-Sems-4
import pygame
import random
import heapq
import os

pygame.init()

map_image = pygame.image.load("MAP.png")
WIDTH, HEIGHT = map_image.get_width(), map_image.get_height()
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Hide and Seek Game")

seek_img = pygame.image.load("seek_arrow.png")
hide_img = pygame.image.load("hide_shield.png")
seek_img = pygame.transform.scale(seek_img, (40, 40))
hide_img = pygame.transform.scale(hide_img, (40, 40))

PASSABLE_COLOR = (150, 150, 150)
MARGIN = 5

map_pixels = pygame.surfarray.array3d(map_image)

# Cek apakah pixel bisa dilewati
def is_passable(x, y):
    if 0 <= x < WIDTH and 0 <= y < HEIGHT:
        r, g, b = map_pixels[x, y]
        return (r, g, b) == PASSABLE_COLOR
    return False

def get_random_edge_position():
    for _ in range(10000):
        side = random.choice(["top", "bottom", "left", "right"])
        if side == "top":
            x, y = random.randint(0, WIDTH - 1), random.randint(0, MARGIN)
        elif side == "bottom":
            x, y = random.randint(0, WIDTH - 1), random.randint(HEIGHT - MARGIN, HEIGHT - 1)
        elif side == "left":
            x, y = random.randint(0, MARGIN), random.randint(0, HEIGHT - 1)
        else:  # right
            x, y = random.randint(WIDTH - MARGIN, WIDTH - 1), random.randint(0, HEIGHT - 1)
        if is_passable(x, y):
            return x, y
    return None

def heuristic(a, b):
    return abs(a[0]-b[0]) + abs(a[1]-b[1])

def a_star(start, goal):
    open_set = []
    heapq.heappush(open_set, (0, start))
    came_from = {}
    g_score = {start: 0}
    while open_set:
        _, current = heapq.heappop(open_set)
        if current == goal:
            path = []
            while current in came_from:
                path.append(current)
                current = came_from[current]
            path.reverse()
            return path
        for dx, dy in [(-1,0),(1,0),(0,-1),(0,1)]:
            neighbor = (current[0]+dx, current[1]+dy)
            if not is_passable(*neighbor):
                continue
            tentative_g = g_score[current] + 1
            if neighbor not in g_score or tentative_g < g_score[neighbor]:
                came_from[neighbor] = current
                g_score[neighbor] = tentative_g
                f = tentative_g + heuristic(neighbor, goal)
                heapq.heappush(open_set, (f, neighbor))
    return []

def draw_seek(pos):
    rect = seek_img.get_rect(center=pos)
    screen.blit(seek_img, rect)

def draw_hide(pos):
    rect = hide_img.get_rect(center=pos)
    screen.blit(hide_img, rect)

font = pygame.font.SysFont(None, 32)
start_button = pygame.Rect(10, 10, 80, 35)
stop_button = pygame.Rect(100, 10, 80, 35)

def draw_buttons():
    pygame.draw.rect(screen, (0, 255, 0), start_button)
    pygame.draw.rect(screen, (255, 0, 0), stop_button)
    screen.blit(font.render("START", True, (255,255,255)), (20, 15))
    screen.blit(font.render("STOP", True, (255,255,255)), (115, 15))

running = True
started = False
path = []
seek_pos = get_random_edge_position()
hide_pos = get_random_edge_position()

while running:
    screen.blit(map_image, (0, 0))
    draw_buttons()

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        elif event.type == pygame.MOUSEBUTTONDOWN:
            if start_button.collidepoint(event.pos):
                started = True
                seek_pos = get_random_edge_position()
                hide_pos = get_random_edge_position()
                if seek_pos and hide_pos:
                    path = a_star(seek_pos, hide_pos)
            elif stop_button.collidepoint(event.pos):
                started = False
                path = []

    if started and path:
        if seek_pos != hide_pos:
            seek_pos = path.pop(0)
        draw_hide(hide_pos)
        draw_seek(seek_pos)
    elif started:
        draw_hide(hide_pos)
        draw_seek(seek_pos)

    pygame.display.flip()
    pygame.time.delay(10)

pygame.quit()
