# Lightning WebSocket Project Summary

## What Worked
We successfully created a system that:
1. Uses Puppeteer to launch a headless browser that loads the Blitzortung.org map
2. Intercepts the WebSocket messages coming from the Blitzortung server
3. Decodes the messages using the same LZW decoding algorithm found in the Blitzortung website code
4. Parses the decoded JSON data to extract lightning strike information
5. Forwards this data to a simple React frontend for display

## What Didn't Work
1. **Direct WebSocket Connection**: Attempting to connect directly to the Blitzortung WebSocket server (ws1.blitzortung.org) from our Node.js server failed consistently. Despite proper headers and connection attempts, we couldn't receive data.

2. **Different Subscription Messages**: We tried various subscription message formats (`{"a":"s","t":1,"r":0}`, etc.) but none successfully established a working data stream without using Puppeteer.

3. **Parsing Without Decoding**: The data from the WebSocket is encoded using a custom LZW compression algorithm. Attempting to parse it directly without decoding resulted in garbage data.

## Data Structure
From our analysis, each lightning strike contains the following key information:

- **time**: Timestamp of the strike (very high precision)
- **lat/lon**: Geographic coordinates of the strike
- **alt**: Altitude (usually 0)
- **pol**: Polarity (0 or 1, indicating positive or negative strike)
- **mds**: Unknown measurement, possibly related to strike strength
- **mcg**: Unknown measurement
- **status**: Status code
- **region**: Region identifier (0-17, corresponds to geographical regions like Europe, North America, etc.)
- **sig**: Array of station objects that detected the strike
- **delay**: Processing delay in seconds
- **lonc/latc**: Adjustment values for coordinates (usually 0)

### Station Object
Each item in the `sig` array represents a station that detected the strike:
- **sta**: Station ID number
- **time**: Time detected (relative to strike time)
- **lat/lon**: Station coordinates
- **alt**: Station altitude
- **status**: Station status code

## Current Status
The system is now successfully capturing lightning strike data in real-time from Blitzortung.org and displaying it in a minimal React interface. The data flow works as follows:

1. Server launches Puppeteer browser
2. Browser loads map.blitzortung.org
3. Server intercepts WebSocket messages 
4. Server decodes and parses data
5. Server forwards data to clients via its own WebSocket
6. React client displays data in a simple table format

This provides a foundation for further development or visualization of real-time lightning data.
