rayon = int(self.inflate_radius / map_resolution)

new_inflated_map = [row[:] for row in map]

height = len(map)

width = len(map[0])

for i in range(height):

for j in range(width):

if map[i][j] == self.MAP_OBSTACLE_VALUE:

for di in range(-rayon, rayon + 1):

for dj in range(-rayon, rayon + 1):

ni = i + di

nj = j + dj

if 0 <= ni < height and 0 <= nj < width:

new_inflated_map[ni][nj] = self.MAP_OBSTACLE_VALUE

  

## UNCOMMENT LINE BELLOW TO TEST YOUR INFLATED MAP

return new_inflated_map