import pygame
import math
import sys
import random
import ctypes
import os

# --- Force System to Ignore OS DPI Scaling (Fixes Window Misalignment) ---
try:
    ctypes.windll.shcore.SetProcessDpiAwareness(2)
except:
    try:
        ctypes.windll.user32.SetProcessDPIAware()
    except:
        pass

# Initialize Pygame
pygame.init()

# --- Big Display Settings ---
WIDTH, HEIGHT = 1920, 1080  
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("The Backrooms: Rapid Fire")
clock = pygame.time.Clock()

# --- Game States ---
game_state = "MENU"

# --- Adjustable Settings ---
sensitivity = 1.0  
difficulty = "NORMAL"  

# --- Map Layout (1 = Yellow Wall, 0 = Hallway) ---
MAP = [
    [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
    [1,0,0,0,0,0,1,0,0,0,0,0,0,1,0,0,0,0,0,1],
    [1,0,1,1,1,0,1,0,1,1,1,1,0,1,0,1,1,1,0,1],
    [1,0,1,0,0,0,0,0,1,0,0,1,0,0,0,1,0,0,0,1],
    [1,0,1,0,1,1,1,1,1,0,0,1,1,1,1,1,0,1,0,1],
    [1,0,0,0,1,0,0,0,0,0,0,0,0,0,0,1,0,1,0,1],
    [1,1,1,0,1,0,1,1,1,1,0,1,1,1,0,1,0,1,0,1],
    [1,0,0,0,0,0,1,0,0,1,0,1,0,0,0,0,0,1,0,1],
    [1,0,1,1,1,1,1,0,0,1,0,1,0,1,1,1,1,1,0,1],
    [1,0,1,0,0,0,0,0,0,1,0,0,0,1,0,0,0,0,0,1],
    [1,0,1,0,1,1,1,1,0,1,1,1,1,1,0,1,1,1,0,1],
    [1,0,0,0,1,0,0,1,0,0,0,0,0,1,0,1,0,0,0,1],
    [1,1,1,1,1,0,0,1,1,1,1,1,0,1,1,1,0,1,1,1],
    [1,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,1],
    [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
]
MAP_W, MAP_H = len(MAP[0]), len(MAP)
TILE_SIZE = 64

# --- PERFORMANCE MODIFIERS ---
STRIDE = 8  
NUM_RAYS = WIDTH // STRIDE  

# --- Z-BUFFER ---
z_buffer = [999999.0] * NUM_RAYS

# --- Game Variables ---
px, py, p_angle = 0, 0, 0
v_lookup = 0.0  
sanity, stamina = 100.0, 100.0
flicker_timer = 0
light_intensity = 1.0

# Score tracking attributes
current_score = 0
kills = 0
survival_ticks = 0
SCORE_FILE = "highscores.txt"

# --- MODIFIED COMBAT ATTRIBUTES (RAPID FIRE) ---
max_ammo = 20       # Upgraded from 6 to 20
ammo = max_ammo
shoot_cooldown = 0
muzzle_flash_timer = 0
screen_shake_timer = 0
is_reloading = False
is_aiming = False

exit_x, exit_y = 10.5 * TILE_SIZE, 11.5 * TILE_SIZE
creatures = []

# --- System Fonts ---
font_title = pygame.font.SysFont("Courier", 64, bold=True)
font_menu = pygame.font.SysFont("Courier", 36, bold=True)
font_hud = pygame.font.SysFont("Courier", 24, bold=True)
font_compass = pygame.font.SysFont("Courier", 28, bold=True)

def load_high_score():
    if not os.path.exists(SCORE_FILE):
        return 0
    try:
        with open(SCORE_FILE, "r") as f:
            return int(f.read().strip())
    except:
        return 0

def save_high_score(score):
    high = load_high_score()
    if score > high:
        with open(SCORE_FILE, "w") as f:
            f.write(str(score))

high_score = load_high_score()

def get_dist(x1, y1, x2, y2):
    return math.hypot(x2 - x1, y2 - y1)

def spawn_creature():
    while True:
        tx = random.randint(1, MAP_W - 2)
        ty = random.randint(1, MAP_H - 2)
        if MAP[ty][tx] == 0:
            cx = tx * TILE_SIZE + TILE_SIZE // 2
            cy = ty * TILE_SIZE + TILE_SIZE // 2
            if get_dist(px, py, cx, cy) > 300:
                base_speed = 1.4 if difficulty == "NORMAL" else (0.8 if difficulty == "EASY" else 2.3)
                return {
                    "x": cx, "y": cy,
                    "speed": base_speed,
                    "state": "WANDERING",
                    "target": [cx, cy],
                    "health": 2,
                    "id": random.randint(0, 100000),
                    "twitch_offset": random.uniform(0, 100)
                }

def get_compass_direction(angle):
    deg = math.degrees(angle) % 360
    if 337.5 <= deg or deg < 22.5:    return "EAST"
    elif 22.5 <= deg < 67.5:         return "SOUTH-EAST"
    elif 67.5 <= deg < 112.5:        return "SOUTH"
    elif 112.5 <= deg < 157.5:       return "SOUTH-WEST"
    elif 157.5 <= deg < 202.5:       return "WEST"
    elif 202.5 <= deg < 247.5:       return "NORTH-WEST"
    elif 247.5 <= deg < 292.5:       return "NORTH"
    else:                            return "NORTH-EAST"

def reset_game():
    global px, py, p_angle, v_lookup, sanity, stamina, flicker_timer, light_intensity, creatures, ammo, shoot_cooldown, is_reloading, current_score, kills, survival_ticks, high_score
    px, py = 1.5 * TILE_SIZE, 1.5 * TILE_SIZE
    p_angle = 0.0
    v_lookup = 0.0  
    sanity = 100.0
    stamina = 100.0
    flicker_timer = 0
    light_intensity = 1.0
    ammo = max_ammo
    shoot_cooldown = 0
    is_reloading = False
    current_score = 0
    kills = 0
    survival_ticks = 0
    high_score = load_high_score()
    
    creatures = []
    for _ in range(3):
        creatures.append(spawn_creature())

reset_game()

# --- Main Game Loop ---
while True:
    clock.tick(144)
    fps_current = int(clock.get_fps())
    
    if shoot_cooldown > 0: shoot_cooldown -= 1
    if muzzle_flash_timer > 0: muzzle_flash_timer -= 1
    if screen_shake_timer > 0: screen_shake_timer -= 1

    if is_reloading and shoot_cooldown == 0:
        ammo = max_ammo
        is_reloading = False

    # ----------------------------------------------------
    # STATE: MAIN MENU
    # ----------------------------------------------------
    if game_state == "MENU":
        pygame.mouse.set_visible(True)
        pygame.event.set_grab(False)
            
        mouse_pos = pygame.mouse.get_pos()
        screen.fill((25, 25, 15)) 
        
        title_lbl = font_title.render("THE BACKROOMS: FAST AUTO", True, (200, 180, 50))
        screen.blit(title_lbl, (WIDTH // 2 - title_lbl.get_width() // 2, HEIGHT // 5))
        
        hi_lbl = font_menu.render(f"ALL-TIME HIGH SCORE: {high_score}", True, (255, 215, 0))
        screen.blit(hi_lbl, (WIDTH // 2 - hi_lbl.get_width() // 2, HEIGHT // 5 + 80))

        start_color = (255, 255, 255) if HEIGHT//2 < mouse_pos[1] < HEIGHT//2 + 50 else (120, 120, 120)
        settings_color = (255, 255, 255) if HEIGHT//2 + 80 < mouse_pos[1] < HEIGHT//2 + 130 else (120, 120, 120)
        exit_color = (255, 255, 255) if HEIGHT//2 + 160 < mouse_pos[1] < HEIGHT//2 + 210 else (120, 120, 120)

        start_btn = font_menu.render("[ START SURVIVAL ]", True, start_color)
        settings_btn = font_menu.render("[ CONTROLS & SETTINGS ]", True, settings_color)
        exit_btn = font_menu.render("[ QUIT ]", True, exit_color)
        
        screen.blit(start_btn, (WIDTH // 2 - start_btn.get_width() // 2, HEIGHT // 2))
        screen.blit(settings_btn, (WIDTH // 2 - settings_btn.get_width() // 2, HEIGHT // 2 + 80))
        screen.blit(exit_btn, (WIDTH // 2 - exit_btn.get_width() // 2, HEIGHT // 2 + 160))
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                if HEIGHT//2 < mouse_pos[1] < HEIGHT//2 + 50:
                    reset_game()
                    pygame.mouse.set_visible(False)
                    pygame.event.set_grab(True) 
                    pygame.mouse.get_rel() 
                    game_state = "PLAYING"
                elif HEIGHT//2 + 80 < mouse_pos[1] < HEIGHT//2 + 130:
                    game_state = "SETTINGS"
                elif HEIGHT//2 + 160 < mouse_pos[1] < HEIGHT//2 + 210:
                    pygame.quit()
                    sys.exit()

    # ----------------------------------------------------
    # STATE: SETTINGS PANEL
    # ----------------------------------------------------
    elif game_state == "SETTINGS":
        mouse_pos = pygame.mouse.get_pos()
        screen.fill((20, 20, 20))
        
        set_title = font_title.render("CONFIGURATION MENU", True, (255, 255, 255))
        screen.blit(set_title, (WIDTH // 2 - set_title.get_width() // 2, HEIGHT // 5))
        
        sens_color = (200, 200, 50) if HEIGHT//2 < mouse_pos[1] < HEIGHT//2 + 40 else (200, 200, 200)
        diff_color = (200, 200, 50) if HEIGHT//2 + 60 < mouse_pos[1] < HEIGHT//2 + 100 else (200, 200, 200)
        back_color = (255, 255, 255) if HEIGHT//2 + 240 < mouse_pos[1] < HEIGHT//2 + 290 else (120, 120, 120)

        sens_lbl = font_menu.render(f"Mouse Sensitivity: < {sensitivity:.1f} >", True, sens_color)
        diff_lbl = font_menu.render(f"Game Difficulty:   < {difficulty} >", True, diff_color)
        
        ctrl1 = font_hud.render("WALK: [W, A, S, D]  |  SPRINT: [HOLD L-SHIFT]", True, (150, 150, 150))
        ctrl2 = font_hud.render("AUTO SHOOT: [HOLD LEFT CLICK]  |  ADS ZOOM AIM: [HOLD RIGHT CLICK]", True, (200, 180, 50))
        back_btn = font_menu.render("[ BACK TO MAIN MENU ]", True, back_color)
        
        screen.blit(sens_lbl, (WIDTH // 2 - sens_lbl.get_width() // 2, HEIGHT // 2))
        screen.blit(diff_lbl, (WIDTH // 2 - diff_lbl.get_width() // 2, HEIGHT // 2 + 60))
        screen.blit(ctrl1, (WIDTH // 2 - ctrl1.get_width() // 2, HEIGHT // 2 + 130))
        screen.blit(ctrl2, (WIDTH // 2 - ctrl2.get_width() // 2, HEIGHT // 2 + 170))
        screen.blit(back_btn, (WIDTH // 2 - back_btn.get_width() // 2, HEIGHT // 2 + 250))
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                if HEIGHT//2 < mouse_pos[1] < HEIGHT//2 + 40:
                    sensitivity = 1.5 if sensitivity == 1.0 else (0.5 if sensitivity == 1.5 else 1.0)
                elif HEIGHT//2 + 60 < mouse_pos[1] < HEIGHT//2 + 100:
                    if difficulty == "EASY": difficulty = "NORMAL"
                    elif difficulty == "NORMAL": difficulty = "HARD"
                    else: difficulty = "EASY"
                elif HEIGHT//2 + 250 < mouse_pos[1] < HEIGHT//2 + 290:
                    game_state = "MENU"

    # ----------------------------------------------------
    # STATE: RUNNING THE 3D GAMEPLAY
    # ----------------------------------------------------
    elif game_state == "PLAYING":
        survival_ticks += 1
        if survival_ticks % 60 == 0:
            current_score += 10

        flicker_timer += 1
        if random.random() < 0.02:
            light_intensity = random.uniform(0.1, 0.45)
        else:
            light_intensity = min(1.0, light_intensity + 0.03)

        shake_x, shake_y = 0, 0
        if screen_shake_timer > 0:
            shake_x = random.randint(-12, 12)
            shake_y = random.randint(-12, 12)

        horizon = (HEIGHT // 2) + int(v_lookup) + shake_y
        ceil_c = (int(40*light_intensity), int(40*light_intensity), int(20*light_intensity))
        floor_c = (int(30*light_intensity), int(25*light_intensity), int(12*light_intensity))
        
        pygame.draw.rect(screen, ceil_c, (0, 0, WIDTH, max(0, horizon)))
        pygame.draw.rect(screen, floor_c, (0, max(0, horizon), WIDTH, HEIGHT))

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                save_high_score(current_score)
                game_state = "MENU"
            if event.type == pygame.KEYDOWN and event.key == pygame.K_r:
                if ammo < max_ammo and not is_reloading:
                    is_reloading = True
                    shoot_cooldown = 35 # Faster reload speed
            if event.type == pygame.MOUSEBUTTONDOWN and pygame.event.get_grab() == False:
                pygame.event.set_grab(True)
                pygame.mouse.get_rel()

        # --- MOUSE INPUT MODULATORS ---
        w_shot_fired = False
        mouse_buttons = pygame.mouse.get_pressed()
        
        is_aiming = mouse_buttons[2] 
        
        if mouse_buttons[0]: 
            if ammo > 0 and shoot_cooldown == 0 and not is_reloading:
                ammo -= 1
                shoot_cooldown = 4     # CRITICAL ADJUSTMENT: Lowered from 16 to 4 frames (Super Fast Auto)
                muzzle_flash_timer = 2
                screen_shake_timer = 3 if is_aiming else 5
                w_shot_fired = True
            elif ammo == 0 and not is_reloading:
                is_reloading = True
                shoot_cooldown = 35

        mouse_dx, mouse_dy = pygame.mouse.get_rel()
        current_sens = sensitivity * 0.45 if is_aiming else sensitivity
        
        p_angle += (mouse_dx + shake_x * 0.1) * 0.0022 * current_sens
        p_angle = p_angle % (2 * math.pi)

        v_lookup -= mouse_dy * 1.5 * current_sens
        v_lookup = max(-540, min(540, v_lookup))

        # Keyboard reading 
        keys = pygame.key.get_pressed()
        speed = 1.8 if is_aiming else 3.5
        if keys[pygame.K_LSHIFT] and stamina > 5 and not is_aiming and (keys[pygame.K_w] or keys[pygame.K_s] or keys[pygame.K_a] or keys[pygame.K_d]):
            speed = 6.0
            stamina = max(0.0, stamina - 0.9)
        else:
            stamina = min(100.0, stamina + 0.25)

        move_x, move_y = 0, 0
        if keys[pygame.K_w]:
            move_x += math.cos(p_angle) * speed
            move_y += math.sin(p_angle) * speed
        if keys[pygame.K_s]:
            move_x -= math.cos(p_angle) * speed
            move_y -= math.sin(p_angle) * speed
        if keys[pygame.K_a]:
            move_x += math.sin(p_angle) * speed
            move_y -= math.cos(p_angle) * speed
        if keys[pygame.K_d]:
            move_x -= math.sin(p_angle) * speed
            move_y += math.cos(p_angle) * speed

        new_px, new_py = px + move_x, py + move_y
        if MAP[int(py / TILE_SIZE)][int(new_px / TILE_SIZE)] == 0: px = new_px
        if MAP[int(new_py / TILE_SIZE)][int(px / TILE_SIZE)] == 0: py = new_py

        # Update creatures
        for c in creatures:
            c_dist = get_dist(px, py, c["x"], c["y"])
            if c_dist < 400:
                sanity = max(0.0, sanity - (550 / (c_dist + 1))) 
                c["state"] = "CHASING"
            else:
                if c["state"] == "CHASING": c["state"] = "WANDERING"

            if c_dist < 24:
                save_high_score(current_score)
                game_state = "GAME_OVER"

            if c["state"] == "WANDERING":
                if get_dist(c["x"], c["y"], c["target"][0], c["target"][1]) < 20:
                    tx, ty = random.randint(1, MAP_W-2), random.randint(1, MAP_H-2)
                    if MAP[ty][tx] == 0:
                        c["target"] = [tx * TILE_SIZE + TILE_SIZE//2, ty * TILE_SIZE + TILE_SIZE//2]
                c_speed = c["speed"]
            else:
                c["target"] = [px, py]
                c_speed = c["speed"] * 2.25

            e_dx = c["target"][0] - c["x"]
            e_dy = c["target"][1] - c["y"]
            e_dist = math.hypot(e_dx, e_dy)
            if e_dist > 5:
                c["x"] += (e_dx / e_dist) * c_speed

        if get_dist(px, py, exit_x, exit_y) < 30:
            current_score += 1000
            save_high_score(current_score)
            game_state = "WIN"

        # --- Dynamic ADS Field-of-View Calculation ---
        DYNAMIC_FOV = (math.pi / 5.14) if is_aiming else (math.pi / 3)
        DYNAMIC_HALF_FOV = DYNAMIC_FOV / 2
        DYNAMIC_ANGLE_STEP = DYNAMIC_FOV / NUM_RAYS

        # --- Raycasting Engine (Populating Z-Buffer) ---
        start_angle = p_angle - DYNAMIC_HALF_FOV
        for ray in range(NUM_RAYS):
            ray_angle = start_angle + ray * DYNAMIC_ANGLE_STEP
            sin_a = math.sin(ray_angle)
            cos_a = math.cos(ray_angle)

            for depth in range(1, 950, 2):  
                tx = px + depth * cos_a
                ty = py + depth * sin_a

                if 0 <= int(ty / TILE_SIZE) < MAP_H and 0 <= int(tx / TILE_SIZE) < MAP_W:
                    if MAP[int(ty / TILE_SIZE)][int(tx / TILE_SIZE)] == 1:
                        corrected_depth = depth * math.cos(p_angle - ray_angle)
                        z_buffer[ray] = corrected_depth

                        proj_coeff = 1500 if is_aiming else 900
                        wall_h = min(HEIGHT * 2, int((TILE_SIZE * proj_coeff) / (corrected_depth + 0.001)))

                        r = int(max(0, min(135, 135 - (corrected_depth / 4.8))) * light_intensity)
                        g = int(max(0, min(125, 125 - (corrected_depth / 4.8))) * light_intensity)
                        b = int(max(0, min(50, 50 - (corrected_depth / 5.8))) * light_intensity)

                        wall_y_center = horizon - (wall_h // 2)
                        pygame.draw.rect(screen, (r, g, b), (ray * STRIDE + shake_x, wall_y_center, STRIDE, wall_h))
                        break

        # --- Sprite Processing Pipeline ---
        rendered_sprites = [('exit', exit_x, exit_y, (140, 0, 0), 40, None)]
        for c in creatures:
            rendered_sprites.append(('monster', c["x"], c["y"], (0,0,0), 65, c))

        rendered_sprites.sort(key=lambda s: get_dist(px, py, s[1], s[2]), reverse=True)

        monster_shot_id = None  
        closest_shot_depth = 999999

        for s_type, sx, sy, s_color, s_size, obj_ref in rendered_sprites:
            s_dx = sx - px
            s_dy = sy - py
            s_ang = math.atan2(s_dy, s_dx) - p_angle
            while s_ang > math.pi: s_ang -= 2 * math.pi
            while s_ang < -math.pi: s_ang += 2 * math.pi

            if abs(s_ang) < DYNAMIC_HALF_FOV:
                s_depth = get_dist(px, py, sx, sy) * math.cos(s_ang)
                if s_depth < 5: s_depth = 5
                
                proj_coeff = 1500 if is_aiming else 900
                s_size_px = min(HEIGHT * 2, int((s_size * proj_coeff) / s_depth))
                sx_scr = int((WIDTH / 2) + (math.tan(s_ang) * (WIDTH / 2)) - (s_size_px / 2)) + shake_x
                sy_scr = int(horizon - (s_size_px / 2))

                sprite_visible = False
                start_ray = max(0, sx_scr // STRIDE)
                end_ray = min(NUM_RAYS - 1, (sx_scr + s_size_px) // STRIDE)
                
                for r_idx in range(start_ray, end_ray + 1):
                    if s_depth < z_buffer[r_idx]:
                        sprite_visible = True
                        break

                if not sprite_visible:
                    continue 

                if s_type == 'monster':
                    t_val = (pygame.time.get_ticks() * 0.008) + obj_ref["twitch_offset"]
                    twitch_mod = math.sin(t_val) * (15 if obj_ref["state"] == "CHASING" else 4)
                    
                    ent_w = s_size_px // 3
                    ent_center_x = sx_scr + (s_size_px // 2) + int(twitch_mod)
                    
                    center_ray = max(0, min(NUM_RAYS - 1, ent_center_x // STRIDE))
                    if s_depth < z_buffer[center_ray]:
                        if ent_center_x - ent_w // 2 <= WIDTH // 2 <= ent_center_x + ent_w // 2 and sy_scr <= horizon <= sy_scr + s_size_px:
                            if s_depth < closest_shot_depth:
                                closest_shot_depth = s_depth
                                monster_shot_id = obj_ref["id"]

                    for r_idx in range(start_ray, end_ray + 1):
                        if s_depth < z_buffer[r_idx]:
                            slice_x = r_idx * STRIDE
                            if ent_center_x - ent_w // 2 <= slice_x <= ent_center_x + ent_w // 2:
                                pygame.draw.rect(screen, (5, 5, 5), (slice_x, sy_scr + s_size_px // 4, STRIDE, int(s_size_px * 0.6)))
                            if ent_center_x - ent_w // 3 <= slice_x <= ent_center_x + ent_w // 3:
                                pygame.draw.rect(screen, (10, 10, 10), (slice_x, sy_scr, STRIDE, ent_w))

                    if random.random() > 0.04 or flicker_timer % 10 < 7:
                        eye_r = (230, 15, 15) if obj_ref["health"] > 1 else (255, 160, 20)
                        eye_y = sy_scr + ent_w // 2
                        
                        eye_left_ray = max(0, min(NUM_RAYS - 1, (ent_center_x - ent_w // 6) // STRIDE))
                        if s_depth < z_buffer[eye_left_ray]:
                            pygame.draw.circle(screen, eye_r, (ent_center_x - ent_w // 6, eye_y), max(2, s_size_px // 50))
                            
                        eye_right_ray = max(0, min(NUM_RAYS - 1, (ent_center_x + ent_w // 8) // STRIDE))
                        if s_depth < z_buffer[eye_right_ray]:
                            pygame.draw.circle(screen, eye_r, (ent_center_x + ent_w // 8, eye_y + int(twitch_mod*0.2)), max(1, s_size_px // 70))
                else:
                    for r_idx in range(start_ray, end_ray + 1):
                        if s_depth < z_buffer[r_idx]:
                            pygame.draw.rect(screen, s_color, (r_idx * STRIDE, sy_scr, STRIDE, s_size_px))

        if w_shot_fired and monster_shot_id is not None:
            for idx, c in enumerate(creatures):
                if c["id"] == monster_shot_id:
                    c["health"] -= 1
                    c["state"] = "CHASING" 
                    if c["health"] <= 0:
                        creatures.pop(idx)
                        creatures.append(spawn_creature())
                        kills += 1
                        current_score += 150
                    break

        if sanity < 50:
            for _ in range(int((50 - sanity) * 45)): 
                screen.set_at((random.randint(0, WIDTH-1), random.randint(0, HEIGHT-1)), (95, 100, 85))

        # --- RADAR MINIMAP ---
        mm_offset_x, mm_offset_y = WIDTH - 250, 50
        mm_tile_scale = 10  
        map_w_px = MAP_W * mm_tile_scale
        map_h_px = MAP_H * mm_tile_scale

        map_surf = pygame.Surface((map_w_px, map_h_px), pygame.SRCALPHA)
        map_surf.fill((15, 15, 15, 160)) 

        for my in range(MAP_H):
            for mx in range(MAP_W):
                if MAP[my][mx] == 1:
                    pygame.draw.rect(map_surf, (120, 110, 55, 190), (mx * mm_tile_scale, my * mm_tile_scale, mm_tile_scale - 1, mm_tile_scale - 1))

        pygame.draw.circle(map_surf, (150, 0, 0, 255), (int((exit_x / TILE_SIZE) * mm_tile_scale), int((exit_y / TILE_SIZE) * mm_tile_scale)), 3)
        for c in creatures:
            pygame.draw.circle(map_surf, (240, 20, 20, 255), (int((c["x"] / TILE_SIZE) * mm_tile_scale), int((c["y"] / TILE_SIZE) * mm_tile_scale)), 2)
        pygame.draw.circle(map_surf, (0, 255, 100, 255), (int((px / TILE_SIZE) * mm_tile_scale), int((py / TILE_SIZE) * mm_tile_scale)), 4)

        screen.blit(map_surf, (mm_offset_x, mm_offset_y))
        pygame.draw.rect(screen, (200, 180, 50), (mm_offset_x - 2, mm_offset_y - 2, map_w_px + 4, map_h_px + 4), 2)

        # --- HUD TEXT OVERLAYS ---
        compass_string = f"HEADING: {get_compass_direction(p_angle)}"
        lbl_comp = font_compass.render(compass_string, True, (235, 235, 235))
        screen.blit(lbl_comp, (WIDTH // 2 - lbl_comp.get_width() // 2, 40))

        # --- GUN MODEL ---
        if is_aiming:
            gun_base_x = WIDTH // 2 - 20 + shake_x
            gun_base_y = int(horizon + (HEIGHT / 6))
            if muzzle_flash_timer > 0:
                pygame.draw.circle(screen, (255, 190, 60), (WIDTH // 2 + shake_x, gun_base_y - 60), random.randint(25, 45))
                pygame.draw.rect(screen, (40, 40, 40), (gun_base_x, gun_base_y, 40, 400))
            else:
                pygame.draw.rect(screen, (28, 28, 28), (gun_base_x, gun_base_y, 40, 400))
        else:
            gun_base_x = WIDTH // 2 - 40 + shake_x
            gun_base_y = int(horizon + (HEIGHT / 4))
            if muzzle_flash_timer > 0:
                pygame.draw.circle(screen, (255, 220, 100), (WIDTH // 2 + shake_x, gun_base_y - 40), random.randint(45, 80))
                pygame.draw.rect(screen, (50, 50, 50), (gun_base_x, gun_base_y + 40, 80, 280)) 
            else:
                pygame.draw.rect(screen, (35, 35, 35), (gun_base_x, gun_base_y, 80, 320))
                pygame.draw.rect(screen, (20, 20, 20), (gun_base_x + 25, gun_base_y - 20, 30, 40))

        if is_aiming:
            pygame.draw.circle(screen, (255, 50, 50), (WIDTH // 2, HEIGHT // 2), 2)
            pygame.draw.circle(screen, (255, 50, 50), (WIDTH // 2, HEIGHT // 2), 16, 1)
        else:
            pygame.draw.circle(screen, (255, 255, 255), (WIDTH // 2, HEIGHT // 2), 4, 1)

        # UI Stats Dashboard
        pygame.draw.rect(screen, (30, 30, 30), (50, 50, 300, 25))
        pygame.draw.rect(screen, (0, 150, 255), (50, 50, int(sanity * 3), 25))
        pygame.draw.rect(screen, (30, 30, 30), (50, 95, 300, 25))
        pygame.draw.rect(screen, (0, 200, 50), (50, 95, int(stamina * 3), 25))

        screen.blit(font_hud.render(f"SANITY: {int(sanity)}%", True, (255, 255, 255)), (370, 50))
        screen.blit(font_hud.render(f"STAMINA: {int(stamina)}%", True, (255, 255, 255)), (370, 95))
        
        ammo_text = f"AMMO: {ammo}/{max_ammo}" if not is_reloading else "AUTO-RELOADING..."
        screen.blit(font_hud.render(ammo_text, True, (255, 50, 50) if ammo <= 1 or is_reloading else (255, 255, 255)), (50, 140))
        
        screen.blit(font_hud.render(f"SCORE: {current_score} | BANISHED: {kills}", True, (255, 215, 0)), (50, 180))
        screen.blit(font_hud.render(f"PERF FPS: {fps_current}", True, (0, 255, 50)), (50, 220))

    # ----------------------------------------------------
    # STATE: GAME OVER / WIN SCREENS
    # ----------------------------------------------------
    elif game_state in ["GAME_OVER", "WIN"]:
        pygame.mouse.set_visible(True)
        pygame.event.set_grab(False)
            
        screen.fill((5, 5, 5) if game_state == "GAME_OVER" else (220, 210, 190))
        
        end_txt = "YOU WERE CONSUMED BY THE CREATURES" if game_state == "GAME_OVER" else "YOU NOCLIPPED OUT OF REALITY SUCCESSFULLY!"
        end_color = (200, 0, 0) if game_state == "GAME_OVER" else (0, 120, 0)
        
        lbl_main = font_title.render(end_txt, True, end_color)
        
        record_str = f"YOUR PERFORMANCE SCORE: {current_score} (BANISHED: {kills})"
        if current_score >= high_score and current_score > 0:
            record_str += " ** NEW ALL-TIME RECORD! **"
            
        lbl_score = font_menu.render(record_str, True, (255, 215, 0))
        lbl_sub = font_menu.render("[ PRESS SPACEBAR TO RETURN TO MAIN MENU ]", True, (150, 150, 150))
        
        screen.blit(lbl_main, (WIDTH // 2 - lbl_main.get_width() // 2, HEIGHT // 2 - 80))
        screen.blit(lbl_score, (WIDTH // 2 - lbl_score.get_width() // 2, HEIGHT // 2))
        screen.blit(lbl_sub, (WIDTH // 2 - lbl_sub.get_width() // 2, HEIGHT // 2 + 80))
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_SPACE:
                game_state = "MENU"

    pygame.display.flip()
