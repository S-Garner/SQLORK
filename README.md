# SQLORK

-- SQL to Create Tables --

-- Create Room Table

CREATE TABLE IF NOT EXISTS Room (
    id SERIAL PRIMARY KEY,
    description TEXT,
    north INT,
    south INT,
    east INT,
    west INT
);

-- Create Player Table

CREATE TABLE IF NOT EXISTS Player (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    health INT DEFAULT 100,
    current_location INT REFERENCES Room(id)
);

-- Create Item Table

CREATE TABLE IF NOT EXISTS Item (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    description TEXT,
    room_id INT REFERENCES Room(id)
);

-- Create Key Table

CREATE TABLE IF NOT EXISTS Key (
    id SERIAL PRIMARY KEY,
    color VARCHAR(50),
    room_id INT REFERENCES Room(id)
);

-- Create Lock Table

CREATE TABLE IF NOT EXISTS Lock (
    id SERIAL PRIMARY KEY,
    color VARCHAR(50),
    room_id INT REFERENCES Room(id)
);

-- Create Enemy Table

CREATE TABLE IF NOT EXISTS Enemy (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    health INT,
    damage INT,
    room_id INT REFERENCES Room(id)
);

-- Create HealthPickup Table

CREATE TABLE IF NOT EXISTS HealthPickup (
    id SERIAL PRIMARY KEY,
    description TEXT,
    room_id INT REFERENCES Room(id)
);

-- Create Inventory Table

CREATE TABLE IF NOT EXISTS Inventory (
    id SERIAL PRIMARY KEY,
    player_id INT REFERENCES Player(id),
    item_id INT REFERENCES Item(id),
    key_id INT REFERENCES Key(id)
);


-- SQL for Stored Procedures --

-- Function to move the player in a direction

CREATE OR REPLACE FUNCTION move_player(player_id INT, direction TEXT) RETURNS TEXT AS $$
DECLARE
    new_room INT;
    current_room INT;
BEGIN
    -- Get the current room of the player
    SELECT current_location INTO current_room FROM Player WHERE id = player_id;

    -- Check the direction and find the new room ID
		
    IF direction = 'north' THEN
        SELECT north INTO new_room FROM Room WHERE id = current_room;
    ELSIF direction = 'south' THEN
        SELECT south INTO new_room FROM Room WHERE id = current_room;
    ELSIF direction = 'east' THEN
        SELECT east INTO new_room FROM Room WHERE id = current_room;
    ELSIF direction = 'west' THEN
        SELECT west INTO new_room FROM Room WHERE id = current_room;
    ELSE
        RETURN 'Invalid direction!';
    END IF;

    -- Check if the new room is NULL
		
    IF new_room IS NULL THEN
        RETURN 'You cannot move in that direction!';
    END IF;

    -- Move player to the new room
		
    UPDATE Player SET current_location = new_room WHERE id = player_id;
    RETURN 'You moved ' || direction || '.';
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION pickup_item(player_id INT, item_name TEXT) RETURNS TEXT AS $$
DECLARE
    item_id INT;
    key_id INT;
    current_room INT;
BEGIN
    -- Get player's current room
    SELECT current_location INTO current_room FROM Player WHERE id = player_id;
    
    -- Check if the player is picking up the goblet
		
    IF item_name = 'Goblet' THEN
        -- End the game
        PERFORM end_game();
        RETURN 'You pick up the goblet and feel a surge of energy surround you. You feel space and time collapse all around you, and you begin to slip through the small gaps that exist between seconds... Insert floppy 2 to continue the adventure.';
    END IF;

    -- Attempt to find the item in the current room
		
    SELECT id INTO item_id FROM Item WHERE name = item_name AND room_id = current_room;
    
    IF FOUND THEN
        -- Add item to inventory
				
        INSERT INTO Inventory (player_id, item_id) VALUES (player_id, item_id);
        
        -- Remove item from room
				
        UPDATE Item SET room_id = NULL WHERE id = item_id;
        
        RETURN 'You picked up the ' || item_name || '.';
    ELSE
        -- Attempt to find the key in the current room
				
        SELECT id INTO key_id FROM Key WHERE color = item_name AND room_id = current_room;
        
        IF FOUND THEN
				
            -- Add key to inventory
						
            INSERT INTO Inventory (player_id, key_id) VALUES (player_id, key_id);
            
            -- Remove key from room
						
            UPDATE Key SET room_id = NULL WHERE id = key_id;
            
            RETURN 'You picked up the ' || item_name || ' key.';
        ELSE
            RETURN 'There is no ' || item_name || ' here.';
        END IF;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Function for combat

CREATE OR REPLACE FUNCTION attack_enemy(player_id INT) RETURNS TEXT AS $$
DECLARE
    enemy_id INT;
    enemy_health INT;
    player_health INT;
    player_damage INT;
    enemy_damage INT;
    current_room INT;
BEGIN

    -- Get the player's current room
		
    SELECT current_location INTO current_room FROM Player WHERE id = player_id;

    -- Find the enemy in the current room
		
    SELECT id, health, damage INTO enemy_id, enemy_health, enemy_damage
    FROM Enemy WHERE room_id = current_room;

    -- Check if there is an enemy
		
    IF enemy_id IS NULL THEN
        RETURN 'There is no enemy here.';
    END IF;

    -- Player attacks the enemy (fixed damage of 10)
		
    player_damage := 10;
    UPDATE Enemy SET health = health - player_damage WHERE id = enemy_id;

    -- Check if the enemy is dead
		
    SELECT health INTO enemy_health FROM Enemy WHERE id = enemy_id;
    IF enemy_health <= 0 THEN
        -- Enemy is dead, remove it
        DELETE FROM Enemy WHERE id = enemy_id;
        RETURN 'You defeated the enemy!';
    END IF;


    -- Enemy attacks back
		
    SELECT health INTO player_health FROM Player WHERE id = player_id;
    UPDATE Player SET health = health - enemy_damage WHERE id = player_id;

    -- Check if the player is dead
		
    SELECT health INTO player_health FROM Player WHERE id = player_id;
    IF player_health <= 0 THEN
        RETURN 'You were defeated by the enemy!';
    END IF;

    RETURN 'You attacked the enemy. Enemy health is now ' || enemy_health || '.';
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION unlock_door(player_id INT, room_id INT) RETURNS TEXT AS $$
DECLARE
    lock_color VARCHAR(50);
    key_id INT;
    inventory_id INT;
BEGIN

    -- Find the lock color for the room
		
    SELECT color INTO lock_color FROM Lock WHERE room_id = room_id;

    IF lock_color IS NULL THEN
        RETURN 'There is no lock on this door.';
    END IF;

    -- Find the key ID based on the lock color
		
    SELECT id INTO key_id FROM Key WHERE color = lock_color;

    -- Check if the player has the key in their inventory
		
    SELECT id INTO inventory_id FROM Inventory WHERE player_id = player_id AND key_id = key_id;

    IF inventory_id IS NULL THEN
        RETURN 'You do not have the correct key to unlock this door.';
    END IF;

    -- Unlock the door (remove the lock)
		
    DELETE FROM Lock WHERE room_id = room_id;

    RETURN 'You unlocked the door.';
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION use_health_pickup(player_id INT) RETURNS TEXT AS $$
DECLARE
    current_room INT;
    health_pickup_id INT;
    new_health INT;
BEGIN

    -- Get player's current room
		
    SELECT current_location INTO current_room FROM Player WHERE id = player_id;

    -- Check for health pickup in the room
    SELECT id INTO health_pickup_id FROM HealthPickup WHERE room_id = current_room;

    IF health_pickup_id IS NULL THEN
        RETURN 'There is no health pickup here.';
    END IF;

    -- Increase player's health but not above 100
    UPDATE Player
    SET health = LEAST(health + 50, 100)
    WHERE id = player_id
    RETURNING health INTO new_health;

    -- Remove the health pickup
    DELETE FROM HealthPickup WHERE id = health_pickup_id;

    RETURN 'You used the health pickup and your health is now ' || new_health || '.';
END;
$$ LANGUAGE plpgsql;

-- Function to quit the game and remove all player data

CREATE OR REPLACE FUNCTION quit_game(player_id INT) RETURNS TEXT AS $$
BEGIN

    -- Delete the player from the game
		
    DELETE FROM Player WHERE id = player_id;

    -- Clean up player-related data
		
    DELETE FROM Inventory WHERE player_id = player_id;

    RETURN 'You have quit the game and all your progress has been deleted.';
END;
$$ LANGUAGE plpgsql;

-- Function to end the game and remove all data

CREATE OR REPLACE FUNCTION end_game() RETURNS VOID AS $$
BEGIN
    DELETE FROM Player;
    DELETE FROM Room;
    DELETE FROM Item;
    DELETE FROM Key;
    DELETE FROM Lock;
    DELETE FROM Enemy;
    DELETE FROM HealthPickup;
    DELETE FROM Inventory;
END;
$$ LANGUAGE plpgsql;

INSERT INTO Room (description, north, south, east, west)
VALUES (
    'You awaken on a damp cobblestone floor. You appear to be in some sort of cell; The room is dark and windowless. A slight draft blows from the east, where you see an unlocked, the doorway leading to a dimly lit hallway. You here foot steps and a distant, low growl eminating from somewhere out of the cell. You are alone.',
    NULL,
    NULL,
    2,
    NULL  
);
INSERT INTO Room (description, north, south, east, west)
VALUES (
    'You stand in a dimly lit hall corridor. You see other cells around you, all locked. You hear the shuffle of feet from the darkness of the cages... You feel the burning glare of treacherous eyes glaring at you from the cold abyss of the dark cells... Luckily, they are closed and sealed, the danger of the unknown hindered for now. To the east you see a door leading to a room bathed in light. You are not alone.',
    NULL,
    NULL,  
    3,
    1   
);
INSERT INTO Room (description, north, south, east, west)
VALUES (
    'Your eyes adjust to the brightness of the well-lit room. You feel a brief relief from the suffocating darkness of the previous chambers, but this is fleeting. You stand in a circular room with a high ceiling. A golden antler chandelier hangs above, bathing the space in a warm, orange-golden glow. This appears to be the room where prisoners are processed before being placed in cells. A pile of treasure and gold—tokens confiscated from prisoners brought into this place—lies untouched. Whoever resides here seems to have no interest in the possessions of the captives. You notice a table in the center of the room, upon which rests a sword and a small, open book with a passage circled in ink. To the north is a wooden door, slightly ajar, leading only to darkness. To the east, a staircase descends into the shadows below. You are alone.',
    NULL,  
    NULL,  
    4, 
    2  
);
INSERT INTO Item (name, description, room_id)
VALUES (
    'Sword',
    'A sharp word with beautiful inscriptions in elvish. It has an image of a great dragon etched in the blade.',
    3
);
INSERT INTO Item (name, description, room_id)
VALUES (
    'Goblet',
    'A brilliant beautiful golden goblet with encrusted jewels of Ruby, Emeralds, and Pearls. There are ancient runes written on it, and almost seems to be older than time itself. There is an ever burning flame of blue fire that seems to create a coherent conversation in an old, dead language... I would suggest not trying to drink from it.',
    32
);
INSERT INTO Key (color, room_id)
VALUES (
    'Yellow',
    17
);
INSERT INTO Key (color, room_id)
VALUES (
    'Pink',
    13
);
INSERT INTO Lock (color, room_id)
VALUES (
    'Yellow',
    20
);
INSERT INTO Lock (color, room_id)
VALUES (
    'Pink',
    27
);
INSERT INTO Enemy (name, health, damage, room_id)
VALUES (
    'Goblin',
    30,
    5,
    6
);
INSERT INTO Enemy (name, health, damage, room_id)
VALUES (
    'Goblin',
    30,
    5,
    12
);
INSERT INTO Enemy (name, health, damage, room_id)
VALUES (
    'Goblin',
    30,
    5,
    10
);
INSERT INTO Enemy (name, health, damage, room_id)
VALUES (
    'Goblin',
    30,
    5,
    17
);
INSERT INTO Enemy (name, health, damage, room_id)
VALUES (
    'Goblin',
    30,
    5,
    39
);
INSERT INTO Enemy (name, health, damage, room_id)
VALUES (
    'Goblin',
    30,
    5,
    37
);
INSERT INTO HealthPickup (description, room_id)
VALUES (
    'A small vial of red liquid that restores 50 health points.',
    13
);
INSERT INTO HealthPickup (description, room_id)
VALUES (
    'A small vial of red liquid that restores 50 health points.',
    38
);
