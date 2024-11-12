# Calculate-RA-Decl-Lat-Long-Distance-Lat-Speed-in-Python-using-Swiss-Epemeris
import swisseph as swe
import math

# Define date and time in local time
year, month, day = 2024, 11, 12
hour, minute, second = 12, 55, 55  # Local time (in hours, minutes, seconds)

# Define UTC offset (e.g., UTC+5:30 would be 5.5)
utc_offset = 5.5  # Adjust this value based on your timezone

# Convert local time to UTC by subtracting the UTC offset
utc_decimal = hour + minute / 60 + second / 3600 - utc_offset

# Adjust the day if UTC time goes below 0 or above 24
if utc_decimal < 0:
    utc_decimal += 24
    day -= 1  # Move to the previous day
elif utc_decimal >= 24:
    utc_decimal -= 24
    day += 1  # Move to the next day

# Define location
longitude = 75.41  # Positive for East, Negative for West
latitude = 17.1241   # Positive for North, Negative for South
altitude = 425         # Altitude in meters above sea level, if known

# Calculate Julian day with the adjusted UTC time
jul_day = swe.julday(year, month, day, utc_decimal)

# Set the observer's location for topocentric calculations
swe.set_topo(longitude, latitude, altitude)

# Calculate the Ascendant (Lagna) with Ayanamsa applied
cusps, ascmc = swe.houses(jul_day, latitude, longitude, b'P')  # Use byte string 'P' for Placidus house system
ascendant = ascmc[0]  # Ascendant is the first element in ascmc
ascendant = ascendant % 360  # Ensure Ascendant is between 0 and 360 degrees

# Apply Ayanamsa to Ascendant position
ayanamsa = swe.get_ayanamsa(jul_day)  # Get the Ayanamsa for the given date
ascendant -= ayanamsa  # Deduct Ayanamsa from the Ascendant position
ascendant = ascendant % 360  # Ensure it stays within 0 to 360 degrees
print("Ascendant (Lahiri Ayanamsha):", ascendant)

# List of celestial objects to calculate positions for
celestial_objects = {
    "Sun": swe.SUN,
    "Moon": swe.MOON,
    "Mars": swe.MARS,
    "Mercury": swe.MERCURY,
    "Jupiter": swe.JUPITER,
    "Venus": swe.VENUS,
    "Saturn": swe.SATURN,
    "Rahu (North Node)": swe.TRUE_NODE,  # True Rahu position
}

# Function to calculate Latitude Speed for a planet
def get_latitude_speed(planet, jul_day):
    # Get the planet's position at the current time
    position_current = swe.calc(jul_day, planet, swe.FLG_SIDEREAL | swe.SIDM_LAHIRI)
    
    # Extract the relevant data  Latitude

    latitude_current = position_current[0][1]  # Latitude is the 2nd element (index 1)

    # Get the planet's position after one day to calculate the change in latitude
    jul_day_next = jul_day + 1  # Increment by 1 day
    position_next = swe.calc(jul_day_next, planet, swe.FLG_SIDEREAL | swe.SIDM_LAHIRI)
    
    # Extract latitude for the next day
    latitude_next = position_next[0][1]
    
    # Calculate the latitude speed (change in latitude per day)
    latitude_speed = (latitude_next - latitude_current)  # Latitude speed in degrees per day
    return latitude_speed

    
# Obliquity of the ecliptic (constant for a given time)
# This value is around 23.43928° (in radians)
epsilon = math.radians(23.43928)

# Function to calculate Right Ascension and Declination for a planet without Ayanamsa
def get_ra_declination_no_ayanamsa(planet, jul_day):
    # Get the planet's position at the current time (Tropical coordinates, no Ayanamsa)
    position = swe.calc(jul_day, planet, swe.FLG_SWIEPH)  # Default ephemeris calculation, no sidereal adjustment
    
    # Extract the relevant data (Longitude, Latitude)
    lambda_ = position[0][0]  # Ecliptic Longitude
    beta = position[0][1]    # Ecliptic Latitude
    
    # Convert from ecliptic to equatorial coordinates (RA and Declination)
    
    # Right Ascension (RA)
    ra = math.atan2(math.sin(math.radians(lambda_)) * math.cos(epsilon) - math.tan(math.radians(beta)) * math.sin(epsilon),
                   math.cos(math.radians(lambda_)))
    ra = math.degrees(ra)  # Convert from radians to degrees
    if ra < 0:
        ra += 360  # Ensure RA is between 0 and 360 degrees
    ra = ra / 15  # Convert to hours (1 hour = 15 degrees)
    
    # Declination (Dec)
    dec = math.asin(math.sin(math.radians(beta)) * math.cos(epsilon) + math.cos(math.radians(beta)) * math.sin(epsilon) * math.sin(math.radians(lambda_)))
    dec = math.degrees(dec)  # Convert from radians to degrees
    
    return ra, dec

# Calculate and print Right Ascension and Declination for each planet without Ayanamsa effect
for name, obj in celestial_objects.items():
    right_ascension, declination = get_ra_declination_no_ayanamsa(obj, jul_day)
    print(f"{name} position (Tropical Coordinates):")
    print(f"  Right Ascension: {right_ascension} hours")
    print(f"  Declination: {declination} degrees")

# Calculate and print positions for each object, latitude speeds for each object 
for name, obj in celestial_objects.items():
    latitude_speed = get_latitude_speed(obj, jul_day)
    position = swe.calc(jul_day, obj, swe.FLG_SIDEREAL | swe.SIDM_LAHIRI)
    print(f"{name} position (Lahiri Ayanamsha):", position)
    print(f"  Latitude Speed: {latitude_speed} degrees per day")

# Calculate Ketu (South Node) by adding 180 degrees to Rahu’s position
rahu_position = swe.calc(jul_day, swe.TRUE_NODE, swe.FLG_SIDEREAL | swe.SIDM_LAHIRI)[0]
ketu_position = (rahu_position[0] + 180) % 360  # Wrap around if it goes over 360 degrees
print("Ketu (South Node) position (Lahiri Ayanamsha):", ketu_position)
