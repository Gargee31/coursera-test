# coursera-test
Repo for coursera tests
-- schema.sql

DROP TABLE IF EXISTS keystrokes;
DROP TABLE IF EXISTS mouse_events;

CREATE TABLE keystrokes (
id INTEGER PRIMARY KEY AUTOINCREMENT,
user_id TEXT NOT NULL,
key TEXT NOT NULL,
press_time REAL NOT NULL,
hold_time REAL NOT NULL,
release_time REAL NOT NULL,
flight_time REAL NOT NULL
);

CREATE TABLE mouse_events (
id INTEGER PRIMARY KEY AUTOINCREMENT,
user_id TEXT NOT NULL,
event_time REAL NOT NULL,
button TEXT NOT NULL,
state TEXT NOT NULL,
x INTEGER NOT NULL,
y INTEGER NOT NULL,
result TEXT NOT NULL
);

Database.py
import sqlite3
from fastapi import FastAPI
from contextlib import contextmanager

DATABASE_PATH = 'User_dynamics.db'

@contextmanager
def get_db():
conn = sqlite3.connect(DATABASE_PATH)
try:
yield conn
finally:
conn.close()

def insert_keystroke_data(data):
with get_db() as conn:
cursor = conn.cursor()
cursor.execute(
'INSERT INTO keystrokes (user_id, key, press_time, hold_time, release_time, flight_time) VALUES (?, ?, ?, ?, ?, ?)',
(data['user_id'], data['key'], data['press_time'], data['hold_time'], data['release_time'], data['flight_time'])
)
conn.commit()

def insert_mouse_data(data):
with get_db() as conn:
cursor = conn.cursor()
cursor.execute(
'INSERT INTO mouse_events (user_id, event_time, button, state, x, y, result) VALUES (?, ?, ?, ?, ?, ?, ?)',
(data['user_id'], data['event_time'], data['button'], data['state'], data['x'], data['y'], data['result'])
)
conn.commit()

import sqlite3
import os

DATABASE_PATH = 'User_dynamics.db'

def init_db():
conn = sqlite3.connect(DATABASE_PATH)
with open('schema.sql') as f:
conn.executescript(f.read())
conn.close()
print(f"Database initialized at {DATABASE_PATH}")

if name == "main":
init_db()

import os
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from dotenv import load_dotenv
from database import insert_keystroke_data, insert_mouse_data

Load environment variables from .env file
load_dotenv()

app = FastAPI()

app.mount("/static", StaticFiles(directory="static"), name="static")

@app.get("/")
async def get():
with open('templates/index.html') as f:
return HTMLResponse(f.read())

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
await websocket.accept()
while True:
data = await websocket.receive_text()
data = eval(data) # Convert the string message to a Python dictionary
if 'key' in data[0]:
insert_keystroke_data(data)
else:
insert_mouse_data(data)
await websocket.send_text(f"Data received: {data}")

if name == "main":
import uvicorn
uvicorn.run(app, host="0.0.0.0", port=8000)

<title>Mouse and Keystroke Tracker</title> <script type="text/javascript" src="/static/js/keystroke_tracker.js"></script> <script type="text/javascript" src="/static/js/mouse_tracker.js"></script>
Mouse and Keystroke Tracker
class KeystrokeTracker {
constructor(userId) {
this.userId = userId;
this.socket = new WebSocket("ws://" + window.location.host + "/ws/" + this.userId);
this.keystrokeData = {};
this.lastReleaseTime = 0;
this.statusElement = document.getElementById('status');
this.isPageVisible = true;
this.initializeWebSocket();
this.addEventListeners();
document.getElementById('user-info').innerHTML = this.userId;
}

generateUserId() {
    return 'user_' + Math.random().toString(36).substring(2, 9);
}

initializeWebSocket() {
    this.socket.onopen = (e) => {
        this.updateStatus("Connected to server");
    };
    this.socket.onmessage = (event) => {
        this.updateStatus(event.data);
    };
    this.socket.onclose = (event) => {
        this.updateStatus("Disconnected from server");
    };
}

updateStatus(message) {
    this.statusElement.innerHTML = message;
}

addEventListeners() {
    document.addEventListener("keydown", this.handleKeyDown.bind(this));
    document.addEventListener("keyup", this.handleKeyUp.bind(this));
    document.addEventListener("visibilitychange", this.handleVisibilityChange.bind(this));
}

handleKeyDown(e) {
    if (!this.isPageVisible) return;
    const pressTime = performance.now();
    const key = e.key;
    if (!this.keystrokeData[key]) {
        this.keystrokeData[key] = {
            key: key,
            pressTime: pressTime,
            holdTime: null,
            releaseTime: null,
            flightTime: pressTime - this.lastReleaseTime
        };
    }
}

handleKeyUp(e) {
    if (!this.isPageVisible) return;
    const releaseTime = performance.now();
    const key = e.key;
    if (this.keystrokeData[key]) {
        const keystrokeEvent = this.keystrokeData[key];
        keystrokeEvent.releaseTime = releaseTime;
        keystrokeEvent.holdTime = releaseTime - keystrokeEvent.pressTime;
        this.lastReleaseTime = releaseTime;
        // Send the completed keystroke data
        this.socket.send(JSON.stringify([keystrokeEvent]));
        // Remove the sent data
        delete this.keystrokeData[key];
    }
}

handleVisibilityChange() {
    this.isPageVisible = !document.hidden;
}
}

document.addEventListener("DOMContentLoaded", () => {
const userId = new KeystrokeTracker().generateUserId();
new KeystrokeTracker(userId);
});

class MouseTracker {
constructor(userId) {
this.userId = userId;
this.socket = new WebSocket("ws://" + window.location.host + "/ws/" + this.userId);
this.mouseData = [];
this.lastReleaseTime = 0;
this.statusElement = document.getElementById('status');
this.isPageVisible = true;
this.dragging = false;
this.lastX = 0;
this.lastY = 0;
this.initializeWebSocket();
this.addEventListeners();
document.getElementById('user-info').innerHTML = this.userId;
}

initializeWebSocket() {
    this.socket.onopen = (e) => {
        this.updateStatus("Connected to server");
    };
    this.socket.onmessage = (event) => {
        this.updateStatus(event.data);
    };
    this.socket.onclose = (event) => {
        this.updateStatus("Disconnected from server");
    };
}

updateStatus(message) {
    this.statusElement.innerHTML = message;
}

addEventListeners() {
    document.addEventListener("mousedown", this.handleMouseDown.bind(this));
    document.addEventListener("mouseup", this.handleMouseUp.bind(this));
    document.addEventListener("mousemove", this.handleMouseMove.bind(this));
    document.addEventListener("wheel", this.handleScroll.bind(this));
}

handleMouseDown(e) {
    if (e.button === 1) return false; // Middle button ignored
    const elapsedTime = performance.now();
    const btn = e.button === 0 ? 'Left' : 'Right';
    const state = 'Pressed';
    const result = 'DD';
    this.dragging = true;
    this.lastX = e.clientX;
    this.lastY = e.clientY;
    this.mouseData.push({
        user_id: this.userId,
        event_time: elapsedTime,
        button: btn,
        state: state,
        x: this.lastX,
        y: this.lastY,
        result: result
    });
    this.sendMouseData();
}

handleMouseUp(e) {
    if (e.button === 1) return false; // Middle button ignored
    const elapsedTime = performance.now();
    const btn = e.button === 0 ? 'Left' : 'Right';
    const state = 'Released';
    const result = 'PC';
    this.dragging = false;
    this.lastX = e.clientX;
    this.lastY = e.clientY;
    this.mouseData.push({
        user_id: this.userId,
        event_time: elapsedTime,
        button: btn,
        state: state,
        x: this.lastX,
        y: this.lastY,
        result: result
    });
    this.sendMouseData();
}

handleMouseMove(e) {
    if (!this.dragging) return;
    const elapsedTime = performance.now();
    const btn = 'Move';
    const state = 'Moving';
    const result = 'DR';
    this.lastX = e.clientX;
    this.lastY = e.clientY;
    this.mouseData.push({
        user_id: this.userId,
        event_time: elapsedTime,
        button: btn,
        state: state,
        x: this.lastX,
        y: this.lastY,
        result: result
    });
    this.sendMouseData();
}

handleScroll(e) {
    const elapsedTime = performance.now();
    const btn = 'Scroll';
    const state = e.deltaY > 0 ? 'Down' : 'Up';
    const result = 'SC';
    this.lastX = e.clientX;
    this.lastY = e.clientY;
    this.mouseData.push({
        user_id: this.userId,
        event_time: elapsedTime,
        button: btn,
        state: state,
        x: this.lastX,
        y: this.lastY,
        result: result
    });
    this.sendMouseData();
}

sendMouseData() {
    if (this.mouseData.length > 0) {
        this.socket.send(JSON.stringify(this.mouseData));
        this.mouseData = [];
    }
}
}

document.addEventListener("DOMContentLoaded", () => {
const userId = 'user_' + Math.random().toString(36).substring(2, 9);
new MouseTracker(userId);
});