# gemini-terraria
gemini terrariaimport pygame
import random
import math

# --- CONFIGURATION ---
SCREEN_WIDTH = 1000
SCREEN_HEIGHT = 700
TILE_SIZE = 20

# World dimensions (in tiles)
WORLD_WIDTH = 150
WORLD_HEIGHT = 100

# Colors (RGB)
SKY_BLUE = (135, 206, 235)
DIRT_BROWN = (134, 84, 57)
GRASS_GREEN = (34, 139, 34)
STONE_GRAY = (105, 105, 105)
PLAYER_COLOR = (230, 50, 50)
AIR_COLOR = (0, 0, 0, 0)

# Tile Types
AIR = 0
GRASS = 1
DIRT = 2
STONE = 3

COLOR_MAP = {
    GRASS: GRASS_GREEN,
    DIRT: DIRT_BROWN,
    STONE: STONE_GRAY
}

# --- WORLD GENERATION ---
def generate_world():
    world = [[AIR for _ in range(WORLD_HEIGHT)] for _ in range(WORLD_WIDTH)]
    
    # Simple terrain elevation using sine waves
    base_height = 40
    amplitude = 10
    frequency = 0.05
    
    for x in range(WORLD_WIDTH):
        # Calculate surface height for column x
        height_offset = int(math.sin(x * frequency) * amplitude + random.randint(-1, 1))
        surface_y = base_height + height_offset
        
        for y in range(WORLD_HEIGHT):
            if y < surface_y:
                world[x][y] = AIR
            elif y == surface_y:
                world[x][y] = GRASS
            elif y < surface_y + 15:
                world[x][y] = DIRT
            else:
                world[x][y] = STONE
                
    return world, base_height - 5

# --- PLAYER CLASS ---
class Player:
    def __init__(self, x_tile, y_tile):
        self.rect = pygame.Rect(x_tile * TILE_SIZE, y_tile * TILE_SIZE, TILE_SIZE * 0.8, TILE_SIZE * 1.8)
        self.vx = 0
        self.vy = 0
        self.speed = 5
        self.jump_power = -10
        self.gravity = 0.5
        self.is_grounded = False

    def handle_input(self):
        keys = pygame.key.get_pressed()
        self.vx = 0
        if keys[pygame.K_a] or keys[pygame.K_LEFT]:
            self.vx = -self.speed
        if keys[pygame.K_d] or keys[pygame.K_RIGHT]:
            self.vx = self.speed
        if (keys[pygame.K_w] or keys[pygame.K_SPACE]) and self.is_grounded:
            self.vy = self.jump_power
            self.is_grounded = False

    def update(self, world):
        # Apply Gravity
        self.vy += self.gravity
        if self.vy > 12:  # Terminal velocity
            self.vy = 12

        # Move Horizontal & Check Collisions
        self.rect.x += self.vx
        self.check_collisions(world, dx=self.vx, dy=0)

        # Move Vertical & Check Collisions
        self.rect.y += self.vy
        self.check_collisions(world, dx=0, dy=self.vy)

    def check_collisions(self, world, dx, dy):
        # Calculate tile boundaries around the player
        start_x = max(0, int(self.rect.left // TILE_SIZE))
        end_x = min(WORLD_WIDTH, int(self.rect.right // TILE_SIZE) + 1)
        start_y = max(0, int(self.rect.top // TILE_SIZE))
        end_y = min(WORLD_HEIGHT, int(self.rect.bottom // TILE_SIZE) + 1)

        for x in range(start_x, end_x):
            for y in range(start_y, end_y):
                if world[x][y] != AIR:
                    tile_rect = pygame.Rect(x * TILE_SIZE, y * TILE_SIZE, TILE_SIZE, TILE_SIZE)
                    if self.rect.colliderect(tile_rect):
                        if dx > 0:  # Moving Right
                            self.rect.right = tile_rect.left
                        if dx < 0:  # Moving Left
                            self.rect.left = tile_rect.right
                        if dy > 0:  # Falling
                            self.rect.bottom = tile_rect.top
                            self.vy = 0
                            self.is_grounded = True
                        if dy < 0:  # Jumping
                            self.rect.top = tile_rect.bottom
                            self.vy = 0

# --- MAIN ENGINE ---
def main():
    pygame.init()
    screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
    pygame.display.set_caption("Terraria Python Clone Base")
    clock = pygame.time.Clock()

    world, spawn_y = generate_world()
    player = Player(WORLD_WIDTH // 2, spawn_y)

    current_tile_selected = DIRT

    running = True
    while running:
        clock.tick(60)

        # --- EVENT HANDLING ---
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            
            # Hotbar Selection (1 = Dirt, 2 = Grass, 3 = Stone)
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_1:
                    current_tile_selected = DIRT
                elif event.key == pygame.K_2:
                    current_tile_selected = GRASS
                elif event.key == pygame.K_3:
                    current_tile_selected = STONE

        # Mouse Input for Breaking/Building Tiles
        mouse_buttons = pygame.mouse.get_pressed()
        if mouse_buttons[0] or mouse_buttons[2]:  # Left or Right click
            mx, my = pygame.mouse.get_pos()
            # Convert screen coordinates to world tile coordinates (accounting for camera offset)
            cam_x = player.rect.centerx - SCREEN_WIDTH // 2
            cam_y = player.rect.centery - SCREEN_HEIGHT // 2
            
            tile_x = int((mx + cam_x) // TILE_SIZE)
            tile_y = int((my + cam_y) // TILE_SIZE)

            if 0 <= tile_x < WORLD_WIDTH and 0 <= tile_y < WORLD_HEIGHT:
                # Left Click -> Break Block
                if mouse_buttons[0]:
                    world[tile_x][tile_y] = AIR
                # Right Click -> Place Selected Block (if not intersecting player)
                elif mouse_buttons[2]:
                    tile_rect = pygame.Rect(tile_x * TILE_SIZE, tile_y * TILE_SIZE, TILE_SIZE, TILE_SIZE)
                    if not player.rect.colliderect(tile_rect):
                        world[tile_x][tile_y] = current_tile_selected

        # --- UPDATE ---
        player.handle_input()
        player.update(world)

        # --- CAMERA OFFSET ---
        cam_x = player.rect.centerx - SCREEN_WIDTH // 2
        cam_y = player.rect.centery - SCREEN_HEIGHT // 2

        # --- RENDER ---
        screen.fill(SKY_BLUE)

        # Calculate visible tiles range to optimize drawing
        min_x = max(0, int(cam_x // TILE_SIZE))
        max_x = min(WORLD_WIDTH, int((cam_x + SCREEN_WIDTH) // TILE_SIZE) + 2)
        min_y = max(0, int(cam_y // TILE_SIZE))
        max_y = min(WORLD_HEIGHT, int((cam_y + SCREEN_HEIGHT) // TILE_SIZE) + 2)

        # Draw Visible World
        for x in range(min_x, max_x):
            for y in range(min_y, max_y):
                tile_type = world[x][y]
                if tile_type != AIR:
                    rect = pygame.Rect(x * TILE_SIZE - cam_x, y * TILE_SIZE - cam_y, TILE_SIZE, TILE_SIZE)
                    pygame.draw.rect(screen, COLOR_MAP[tile_type], rect)

        # Draw Player
        player_screen_rect = pygame.Rect(
            player.rect.x - cam_x,
            player.rect.y - cam_y,
            player.rect.width,
            player.rect.height
        )
        pygame.draw.rect(screen, PLAYER_COLOR, player_screen_rect)

        # Simple UI / Instructions
        font = pygame.font.SysFont(None, 24)
        info_text = font.render(f"Keys: A/D (Move), W/Space (Jump) | Left Click: Mine | Right Click: Place | 1-3: Choose Block", True, (0, 0, 0))
        screen.blit(info_text, (10, 10))

        pygame.display.flip()

    pygame.quit()

if __name__ == "__main__":
    main()
