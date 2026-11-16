# BAKHMUT-EARS-TO-DIE-2
mkdir -p /home/claude/BAKHMUT-EARS-TO-DIE-2 && cd /home/claude/BAKHMUT-EARS-TO-DIE-2 && mkdir -p project scenes scripts assets audio textures characters weapons vehicles bosses && for d in project assets audio textures; do touch $d/.gitkeep; done

cat > project.godot << 'EOF'
; BAKHMUT: EARS TO DIE 2 — Godot 4.2+
config_version=5

[application]
config/name="BAKHMUT: EARS TO DIE 2"
run/main_scene="res://scenes/main.tscn"
config/features=PackedStringArray("4.2", "Forward Plus")

[autoload]
Game="*res://scripts/game.gd"
Sfx="*res://scripts/sfx.gd"

[display]
window/size/viewport_width=1280
window/size/viewport_height=720
window/stretch/mode="canvas_items"

[rendering]
environment/defaults/default_clear_color=Color(0.02, 0.02, 0.04, 1)
EOF

cat > scenes/main.tscn << 'EOF'
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/main.gd" id="1"]

[node name="Main" type="Node3D"]
script = ExtResource("1")
EOF

cat > .gitignore << 'EOF'
.godot/
.import/
export.cfg
export_credentials.cfg
*.translation
build/
*.tmp
EOF

cat > LICENSE << 'EOF'
MIT License

Copyright (c) 2026 BAKHMUT: EARS TO DIE 2 authors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
EOF

cat > scripts/gfx.gd << 'EOF'
extends RefCounted
## Процедурна графіка: меші, колізії, 3D-текст (без зовнішніх асетів).

static func mat(c: Color, emit := false) -> StandardMaterial3D:
	var m := StandardMaterial3D.new()
	m.albedo_color = c
	m.roughness = 0.9
	if emit:
		m.emission_enabled = true
		m.emission = c
		m.emission_energy_multiplier = 1.6
	return m

static func _mi(mesh: Mesh, c: Color, emit: bool) -> MeshInstance3D:
	var mi := MeshInstance3D.new()
	mi.mesh = mesh
	mi.material_override = mat(c, emit)
	return mi

static func box(size: Vector3, c: Color, emit := false) -> MeshInstance3D:
	var m := BoxMesh.new()
	m.size = size
	return _mi(m, c, emit)

static func sphere(r: float, c: Color, emit := false) -> MeshInstance3D:
	var m := SphereMesh.new()
	m.radius = r
	m.height = r * 2.0
	return _mi(m, c, emit)

static func cyl(r: float, h: float, c: Color, emit := false) -> MeshInstance3D:
	var m := CylinderMesh.new()
	m.top_radius = r
	m.bottom_radius = r
	m.height = h
	return _mi(m, c, emit)

static func capsule(r: float, h: float, c: Color) -> MeshInstance3D:
	var m := CapsuleMesh.new()
	m.radius = r
	m.height = h
	return _mi(m, c, false)

static func solid(parent: Node, pos: Vector3, size: Vector3, c: Color) -> StaticBody3D:
	var sb := StaticBody3D.new()
	var cs := CollisionShape3D.new()
	var sh := BoxShape3D.new()
	sh.size = size
	cs.shape = sh
	sb.add_child(cs)
	sb.add_child(box(size, c))
	sb.position = pos
	parent.add_child(sb)
	return sb

static func text3d(parent: Node, text: String, pos: Vector3, size := 32, col := Color.WHITE) -> Label3D:
	var l := Label3D.new()
	l.text = text
	l.font_size = size
	l.pixel_size = 0.01
	l.billboard = BaseMaterial3D.BILLBOARD_ENABLED
	l.no_depth_test = true
	l.modulate = col
	l.outline_size = 10
	l.position = pos
	parent.add_child(l)
	return l

static func popup(text: String, pos: Vector3, col := Color.WHITE, size := 40) -> void:
	if not is_instance_valid(Game.main):
		return
	var l := text3d(Game.main, text, pos, size, col)
	var tw := l.create_tween()
	tw.tween_property(l, "position:y", pos.y + 1.5, 0.9)
	tw.parallel().tween_property(l, "modulate:a", 0.0, 0.9)
	tw.tween_callback(l.queue_free)
EOF

cat > scripts/secret.gd << 'EOF'
extends Node3D
## Схованка / прихований люк. Невидима, поки її не знайшов гравець або собака.

var revealed := false

func _ready() -> void:
	add_to_group("secret")
	visible = false

func reveal() -> void:
	if revealed:
		return
	revealed = true
	visible = true
	Sfx.play("pickup")
	Game.say("Знайдено схованку!", Color.CYAN)
EOF

cat > scripts/game.gd << 'EOF'
extends Node
## Глобальний стан гри (autoload "Game").

signal message(text: String, color: Color)

const SKILL_DEFS := [
	["hp1", "Здоров'я I", "", "+25 макс. HP"],
	["hp2", "Здоров'я II", "hp1", "+25 HP, парирування лікує на 5 більше"],
	["st1", "Витривалість I", "", "+25 витривалості"],
	["st2", "Витривалість II", "st1", "+25 витр., ширше вікно парирування"],
	["dm1", "Сила I", "", "+15% шкоди"],
	["dm2", "Сила II", "dm1", "+20% шкоди"],
	["cr1", "Броня Жигулів", "", "+50 міцності авто"],
	["cr2", "Великий бак", "cr1", "+40 пального"],
	["gn1", "Огіркова гармата+", "dm1", "x1.5 шкода, швидше, +40 огірків"],
	["bd1", "Бідон-гарпун+", "dm1", "+60% шкоди бідона"],
]

var gold := 50
var xp := 0
var level := 1
var skill_points := 0
var max_hp := 100.0
var max_stam := 100.0
var dmg_mult := 1.0
var parry_heal := 10.0
var parry_window := 0.28
var car_max_hp := 100.0
var car_fuel_max := 60.0
var cannon_lvl := 1
var bidon_bonus := 0.0
var items := {}
var skills := {}
var kills := 0
var boss1_dead := false
var boss2_dead := false
var menu := ""
var captured := false
var player = null
var car = null
var dog = null
var main = null

func _ready() -> void:
	reset()

func reset() -> void:
	gold = 50
	xp = 0
	level = 1
	skill_points = 0
	max_hp = 100.0
	max_stam = 100.0
	dmg_mult = 1.0
	parry_heal = 10.0
	parry_window = 0.28
	car_max_hp = 100.0
	car_fuel_max = 60.0
	cannon_lvl = 1
	bidon_bonus = 0.0
	items = {"Огірки": 20, "Аптечка": 1, "Пляшка": 4, "Монета": 12, "Мотлох": 2, "Рідкісний ресурс": 0}
	skills = {}
	kills = 0
	boss1_dead = false
	boss2_dead = false
	menu = ""
	captured = false
	player = null
	car = null
	dog = null
	main = null

func say(t: String, c := Color.WHITE) -> void:
	message.emit(t, c)

func add_item(n: String, c := 1) -> void:
	items[n] = int(items.get(n, 0)) + c

func take_item(n: String, c := 1) -> bool:
	if int(items.get(n, 0)) >= c:
		items[n] = int(items[n]) - c
		return true
	return false

func cuke_cap() -> int:
	return 40 + 40 * (cannon_lvl - 1)

func xp_needed() -> int:
	return 60 * level

func add_xp(n: int) -> void:
	xp += n
	while xp >= xp_needed():
		xp -= xp_needed()
		level += 1
		skill_points += 1
		say("РІВЕНЬ %d! +1 очко навичок (K)" % level, Color.YELLOW)
		Sfx.play("level")
		if player != null:
			player.hp = minf(player.hp + 25.0, max_hp)

func skill_name(id: String) -> String:
	for d in SKILL_DEFS:
		if d[0] == id:
			return d[1]
	return id

func skill_menu_text() -> String:
	var s := "ДЕРЕВО НАВИЧОК — очки: %d    (K / E — закрити)\n\n" % skill_points
	for i in SKILL_DEFS.size():
		var d: Array = SKILL_DEFS[i]
		var st := ""
		if skills.has(d[0]):
			st = "[КУПЛЕНО]"
		elif d[2] != "" and not skills.has(d[2]):
			st = "[потрібно: %s]" % skill_name(d[2])
		else:
			st = "[1 очко]"
		s += "%d. %s — %s %s\n" % [(i + 1) % 10, d[1], d[3], st]
	return s

func buy_skill_idx(i: int) -> void:
	if i < 0 or i >= SKILL_DEFS.size():
		return
	var d: Array = SKILL_DEFS[i]
	var id: String = d[0]
	if skills.has(id):
		say("Вже куплено.", Color.ORANGE)
		return
	if d[2] != "" and not skills.has(d[2]):
		say("Спочатку: " + skill_name(d[2]), Color.ORANGE)
		return
	if skill_points <= 0:
		say("Немає очок навичок.", Color.ORANGE)
		return
	skill_points -= 1
	skills[id] = true
	match id:
		"hp1":
			max_hp += 25.0
			player.hp += 25.0
		"hp2":
			max_hp += 25.0
			parry_heal += 5.0
			player.hp += 25.0
		"st1":
			max_stam += 25.0
		"st2":
			max_stam += 25.0
			parry_window += 0.07
		"dm1":
			dmg_mult += 0.15
		"dm2":
			dmg_mult += 0.20
		"cr1":
			car_max_hp += 50.0
			car.hp += 50.0
		"cr2":
			car_fuel_max += 40.0
		"gn1":
			cannon_lvl = 2
		"bd1":
			bidon_bonus += 0.6
	Sfx.play("level")
	say("Навичка: " + d[1], Color.YELLOW)
EOF

cat > scripts/sfx.gd << 'EOF'
extends Node
## Процедурний звук (autoload "Sfx"): усі звуки й музика генеруються кодом.

const RATE := 22050
var cache := {}
var mp: AudioStreamPlayer
var cur := ""

func _ready() -> void:
	mp = AudioStreamPlayer.new()
	mp.volume_db = -13.0
	add_child(mp)

func _gen(freq: float, dur: float, vol := 0.5, wave := "sq", slide := 0.0, env := 1.0) -> PackedByteArray:
	var n := int(RATE * dur)
	var data := PackedByteArray()
	data.resize(n * 2)
	var ph := 0.0
	for i in n:
		var t := float(i) / float(n)
		ph += maxf(20.0, freq + slide * t) / float(RATE)
		var s := 0.0
		match wave:
			"sq":
				s = 1.0 if fmod(ph, 1.0) < 0.5 else -1.0
			"sin":
				s = sin(ph * TAU)
			"saw":
				s = fmod(ph, 1.0) * 2.0 - 1.0
			_:
				s = randf() * 2.0 - 1.0
		s *= vol * pow(1.0 - t, env)
		data.encode_s16(i * 2, int(clampf(s, -1.0, 1.0) * 32000.0))
	return data

func _wav(data: PackedByteArray, loop := false) -> AudioStreamWAV:
	var w := AudioStreamWAV.new()
	w.format = AudioStreamWAV.FORMAT_16_BITS
	w.mix_rate = RATE
	w.stereo = false
	w.data = data
	if loop:
		w.loop_mode = AudioStreamWAV.LOOP_FORWARD
		w.loop_begin = 0
		w.loop_end = data.size() / 2
	return w

func _build(n: String) -> AudioStreamWAV:
	var d := PackedByteArray()
	match n:
		"hit":
			d = _gen(0, 0.12, 0.55, "n", 0.0, 2.0)
		"swing":
			d = _gen(0, 0.09, 0.22, "n", 0.0, 1.0)
		"heavy":
			d = _gen(140, 0.28, 0.5, "sq", -90.0, 1.5)
		"parry":
			d = _gen(880, 0.07, 0.6, "sin", 0.0, 0.5) + _gen(1320, 0.24, 0.6, "sin", 0.0, 1.5)
		"block":
			d = _gen(200, 0.09, 0.45, "sq", -60.0, 1.0)
		"hurt":
			d = _gen(220, 0.25, 0.5, "saw", -140.0, 1.0)
		"dodge":
			d = _gen(0, 0.1, 0.2, "n", 0.0, 0.8)
		"shoot":
			d = _gen(650, 0.1, 0.4, "sq", -450.0, 1.0)
		"bidon":
			d = _gen(300, 0.2, 0.5, "saw", 300.0, 1.0) + _gen(520, 0.1, 0.4, "sq", -100.0, 1.0)
		"whip":
			d = _gen(0, 0.12, 0.4, "n", 0.0, 1.0) + _gen(1600, 0.06, 0.5, "sin", -1300.0, 1.0)
		"honk":
			d = _gen(330, 0.45, 0.45, "sq", 0.0, 0.3) + _gen(262, 0.45, 0.45, "sq", 0.0, 0.3)
		"level":
			d = _gen(523, 0.1, 0.5, "sin") + _gen(659, 0.1, 0.5, "sin") + _gen(784, 0.1, 0.5, "sin") + _gen(1047, 0.3, 0.5, "sin")
		"coin":
			d = _gen(1500, 0.05, 0.4, "sin") + _gen(2000, 0.12, 0.4, "sin")
		"bottle":
			d = _gen(0, 0.28, 0.5, "n", 0.0, 1.5)
		"bark":
			d = _gen(420, 0.13, 0.5, "saw", -180.0, 1.0)
		"boom":
			d = _gen(0, 0.5, 0.7, "n", 0.0, 2.5)
		"pickup":
			d = _gen(700, 0.1, 0.4, "sin", 500.0, 1.0)
		"death":
			d = _gen(300, 0.5, 0.5, "sq", -250.0, 1.0)
		_:
			d = _gen(440, 0.08, 0.3, "sin")
	return _wav(d)

func play(n: String, vol_db := 0.0) -> void:
	if not cache.has(n):
		cache[n] = _build(n)
	var p := AudioStreamPlayer.new()
	p.stream = cache[n]
	p.volume_db = vol_db
	add_child(p)
	p.play()
	p.finished.connect(p.queue_free)

func _melody(notes: Array, dur: float, wave: String, vol: float) -> PackedByteArray:
	var d := PackedByteArray()
	for f in notes:
		d += _gen(float(f), dur, vol, wave, 0.0, 0.6)
	return d

func music(kind: String) -> void:
	if kind == cur:
		return
	cur = kind
	if kind == "":
		mp.stop()
		return
	var key := "m_" + kind
	if not cache.has(key):
		match kind:
			"boss1":
				cache[key] = _wav(_melody([220, 262, 330, 220, 247, 294, 370, 247, 196, 233, 294, 196, 185, 220, 277, 185], 0.2, "sq", 0.22), true)
			"boss2":
				cache[key] = _wav(_melody([220, 233, 220, 208, 220, 233, 247, 233, 165, 175, 165, 156, 165, 175, 185, 175], 0.15, "saw", 0.2), true)
			_:
				cache[key] = _wav(_melody([110, 110, 131, 98, 110, 110, 147, 98], 0.7, "sin", 0.4), true)
	mp.stream = cache[key]
	mp.play()
EOF
echo ok