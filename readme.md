# Edge to cloud

This project demonstrates an **Edge-to-Cloud** architecture where the **Raspberry Pi Pico W** acts as the **edge** device and communicates with a **Node.js Express.js** backend (**cloud**) using HTTP requests. The Pico W collects data (e.g., sensor readings) and sends it to the cloud via HTTP GET requests. The cloud backend receives this data through a REST API endpoint that accepts a query parameter `value`.

## Components

- **Edge**: Raspberry Pi Pico W running MicroPython
  - Collects data (e.g., from sensors) and sends it to the cloud.
  - Communicates with the cloud using HTTP GET requests via the `urequests` module.
  
- **Cloud**: Node.js Express.js REST API
  - Provides a GET endpoint (`/data`) that accepts the `value` parameter.
  - Logs or processes data sent from the Pico W.

## Project Overview

1. **Raspberry Pi Pico W** collects some data (e.g., sensor readings or static data) and sends it to the cloud using an HTTP GET request.
2. The **Node.js Express.js API** listens for incoming GET requests on the `/data` endpoint and processes the query parameter `value`, which contains the data sent from the edge device.
3. The API logs or stores the data for further analysis or processing.

For more details, see `readme.md` files at `cloud` and `edge_v2` directories.

## Conclusion

This project demonstrates how to set up an **Edge-to-Cloud** communication system where the **Raspberry Pi Pico W** (Edge device) sends data via HTTP GET requests to a **Node.js Express.js REST API** (Cloud). The Pico W collects or generates data (e.g., sensor readings) and transmits it to the cloud for further processing, enabling IoT use cases such as remote monitoring, data logging, and automation.

## Next Steps

- Expand the functionality to use **POST** requests for sending larger datasets.
- Add **sensor integration** on the Pico W to collect real-world data.
- Implement **data storage** on the cloud side (e.g., using a database like InfluxDB).
- Add **error handling** and **retries** for robust communication.

## License

This project is open-source and available under the MIT License.

## Acknowledgments

- [MicroPython documentation](https://docs.micropython.org)
- [Node.js v20.x documentation](https://nodejs.org/docs/latest-v20.x/api/index.html)
- [Express.js](https://expressjs.com/)

---

This `readme.md` provides a comprehensive guide on how to set up edge-to-cloud communication using **Pico W** and **Node.js Express.js** for simple IoT applications.
