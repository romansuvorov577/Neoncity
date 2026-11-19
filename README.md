# Neoncity
from kivy.app import App
from kivy.uix.widget import Widget
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.button import Button
from kivy.uix.label import Label
from kivy.clock import Clock
from kivy.graphics import Color, Rectangle, Ellipse, Line
from kivy.core.window import Window
from kivy.metrics import dp
import random, math

WORLD_W, WORLD_H = 1800, 1400
ROAD = 150
BLOCK = 300

class CityGame(Widget):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.player = [WORLD_W/2, WORLD_H/2]
        self.angle = 0
        self.speed = 0
        self.coins = 0
        self.deliveries = 0
        self.message = "Собирай жетоны и выполняй доставки!"
        self.keys = set()
        self.touch_dirs = set()
        self.buildings = []
        self.tokens = []
        self.cars = []
        self._make_city()
        self.bind(size=lambda *_: self.draw())
        Window.bind(on_key_down=self.on_key_down, on_key_up=self.on_key_up)
        Clock.schedule_interval(self.update, 1/60)

    def _make_city(self):
        random.seed(8)
        # A grid of blocks with streets between them.
        for x in range(70, WORLD_W-180, BLOCK):
            for y in range(70, WORLD_H-180, BLOCK):
                w = random.randint(105, 145)
                h = random.randint(105, 145)
                bx = x + random.randint(15, 85)
                by = y + random.randint(15, 85)
                self.buildings.append((bx, by, w, h, random.choice([
                    (0.22,0.36,0.48), (0.34,0.30,0.43),
                    (0.30,0.42,0.36), (0.48,0.34,0.24)])))
        for _ in range(24):
            self.tokens.append([random.randint(90,WORLD_W-90), random.randint(90,WORLD_H-90)])
        for i in range(12):
            self.cars.append([random.randint(100,WORLD_W-100), random.choice([ROAD/2, WORLD_H-ROAD/2]), random.choice([-1,1])])

    def on_key_down(self, window, key, scancode, codepoint, modifiers):
        self.keys.add(key)
        return True

    def on_key_up(self, window, key, *args):
        self.keys.discard(key)
        return True

    def set_dir(self, direction, active):
        if active: self.touch_dirs.add(direction)
        else: self.touch_dirs.discard(direction)

    def collides(self, x, y):
        r = 18
        if x < r or y < r or x > WORLD_W-r or y > WORLD_H-r: return True
        for bx,by,w,h,_ in self.buildings:
            if bx-r < x < bx+w+r and by-r < y < by+h+r: return True
        return False

    def update(self, dt):
        forward = ('up' in self.touch_dirs or 273 in self.keys or 119 in self.keys)
        backward = ('down' in self.touch_dirs or 274 in self.keys or 115 in self.keys)
        left = ('left' in self.touch_dirs or 276 in self.keys or 97 in self.keys)
        right = ('right' in self.touch_dirs or 275 in self.keys or 100 in self.keys)
        if forward: self.speed = min(self.speed + 230*dt, 310)
        elif backward: self.speed = max(self.speed - 260*dt, -120)
        else: self.speed *= max(0, 1-2.2*dt)
        if left: self.angle += (125 + abs(self.speed)*0.18)*dt
        if right: self.angle -= (125 + abs(self.speed)*0.18)*dt
        rad = math.radians(self.angle)
        nx = self.player[0] + math.cos(rad)*self.speed*dt
        ny = self.player[1] + math.sin(rad)*self.speed*dt
        if not self.collides(nx, ny):
            self.player = [nx, ny]
        else:
            self.speed *= -0.22
        for token in self.tokens[:]:
            if math.hypot(token[0]-self.player[0], token[1]-self.player[1]) < 35:
                self.tokens.remove(token); self.coins += 1
                self.message = f"Жетон найден! Всего: {self.coins}"
        if not self.tokens:
            self.tokens = [[random.randint(90,WORLD_W-90), random.randint(90,WORLD_H-90)] for _ in range(24)]
            self.message = "Новая партия жетонов появилась в городе!"
        # Decorative traffic moves along the lower avenue.
        for car in self.cars:
            car[0] += car[2]*70*dt
            if car[0] < 30: car[0] = WORLD_W-30
            if car[0] > WORLD_W-30: car[0] = 30
        self.draw()

    def draw(self):
        self.canvas.clear()
        if self.width <= 0 or self.height <= 0: return
        sx = self.width/WORLD_W
        sy = self.height/WORLD_H
        scale = min(sx, sy)
        ox = (self.width-WORLD_W*scale)/2
        oy = (self.height-WORLD_H*scale)/2
        px, py = self.player
        camx = max(0, min(WORLD_W-self.width/scale, px-self.width/scale/2))
        camy = max(0, min(WORLD_H-self.height/scale, py-self.height/scale/2))
        def pos(x,y): return (ox+(x-camx)*scale, oy+(y-camy)*scale)
        with self.canvas:
            Color(0.10,0.15,0.17); Rectangle(pos=self.pos, size=self.size)
            # Grass and road grid
            Color(0.12,0.22,0.19); Rectangle(pos=(ox-camx*scale,oy-camy*scale), size=(WORLD_W*scale,WORLD_H*scale))
            Color(0.20,0.22,0.23)
            for x in range(0,WORLD_W+1,BLOCK):
                p=pos(x-ROAD/2,0); Rectangle(pos=p,size=(ROAD*scale,WORLD_H*scale))
            for y in range(0,WORLD_H+1,BLOCK):
                p=pos(0,y-ROAD/2); Rectangle(pos=p,size=(WORLD_W*scale,ROAD*scale))
            Color(0.82,0.73,0.42)
            for x in range(0,WORLD_W+1,BLOCK):
                p=pos(x-2,0); Line(points=[p[0],p[1],p[0],p[1]+WORLD_H*scale],width=1)
            for y in range(0,WORLD_H+1,BLOCK):
                p=pos(0,y-2); Line(points=[p[0],p[1],p[0]+WORLD_W*scale,p[1]],width=1)
            # Buildings
            for bx,by,w,h,c in self.buildings:
                Color(*c); Rectangle(pos=pos(bx,by), size=(w*scale,h*scale))
                Color(0.65,0.75,0.76,0.45)
                for wx in range(12,int(w)-8,28):
                    for wy in range(12,int(h)-8,28):
                        q=pos(bx+wx,by+wy); Rectangle(pos=q,size=(9*scale,8*scale))
            # Collectibles
            Color(1,0.78,0.16)
            for tx,ty in self.tokens:
                q=pos(tx,ty); Ellipse(pos=(q[0]-8*scale,q[1]-8*scale),size=(16*scale,16*scale))
            # Traffic
            for cx,cy,d in self.cars:
                q=pos(cx,cy); Color(0.2,0.65,0.95)
                Rectangle(pos=(q[0]-13*scale,q[1]-7*scale),size=(26*scale,14*scale))
            # Player car
            q=pos(px,py)
            Color(0.95,0.27,0.18)
            with self.canvas:
                pass
            # Rotated-looking car using a line and circle marker
            rad=math.radians(self.angle)
            ex=q[0]+math.cos(rad)*22*scale; ey=q[1]+math.sin(rad)*22*scale
            Color(1,0.95,0.85); Line(points=[q[0],q[1],ex,ey],width=max(1,3*scale))
            Color(0.95,0.25,0.16); Ellipse(pos=(q[0]-11*scale,q[1]-8*scale),size=(22*scale,16*scale))
            Color(1,1,1); Ellipse(pos=(ex-3*scale,ey-3*scale),size=(6*scale,6*scale))

class NeonCityApp(App):
    def build(self):
        root = BoxLayout(orientation='vertical', spacing=dp(4), padding=dp(5))
        hud = BoxLayout(size_hint_y=None, height=dp(42))
        self.game = CityGame()
        self.status = Label(text="NEON CITY  •  Жетоны: 0", bold=True, font_size='15sp')
        hud.add_widget(self.status)
        root.add_widget(hud)
        root.add_widget(self.game)
        info = Label(text="Управление: кнопки ниже или WASD / стрелки. Собирай золотые жетоны.", size_hint_y=None, height=dp(28), font_size='11sp')
        root.add_widget(info)
        controls = BoxLayout(size_hint_y=None, height=dp(76), spacing=dp(5))
        for label, direction in [('◀','left'),('▲','up'),('▼','down'),('▶','right')]:
            b=Button(text=label, font_size='24sp', background_normal='', background_color=(0.12,0.34,0.42,1))
            b.bind(on_press=lambda btn,d=direction:self.game.set_dir(d,True))
            b.bind(on_release=lambda btn,d=direction:self.game.set_dir(d,False))
            controls.add_widget(b)
        root.add_widget(controls)
        Clock.schedule_interval(self.refresh_hud, .2)
        return root

    def refresh_hud(self, dt):
        if hasattr(self, 'game'):
            self.status.text = f"NEON CITY  •  Жетоны: {self.game.coins}  •  {self.game.message}"

if __name__ == '__main__':
    NeonCityApp().run()
