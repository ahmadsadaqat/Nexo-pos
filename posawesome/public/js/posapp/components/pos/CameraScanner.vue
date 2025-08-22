<template>
	<v-dialog v-model="scannerDialog" max-width="600px" persistent="false">
		<v-card>
			<v-card-title class="text-h5 text-primary d-flex align-center">
				<v-icon class="mr-2" size="large">mdi-camera</v-icon>
				{{ __("Scan QR Code/Barcode") }}
				<v-chip class="ml-2" size="small" color="primary">
					{{ scanType === "Both" ? "Auto Detect" : scanType }}
				</v-chip>
				<v-spacer></v-spacer>
				<v-btn
					icon="mdi-close"
					@click.stop="stopScanning"
					color="error"
					variant="text"
					size="large"
					:title="__('Close Scanner')"
				></v-btn>
			</v-card-title>

			<v-card-text class="pa-0">
				<div v-if="!cameraPermissionDenied">
					<!-- Scanner container -->
					<div class="scanner-container" v-if="isScanning && scannerDialog">
						<qrcode-stream
							:formats="readerFormats"
							:torch="torchActive"
							:camera="
								selectedDeviceId
									? { deviceId: selectedDeviceId, exact: selectedDeviceId }
									: 'auto'
							"
							@detect="onDetect"
							@error="onError"
							@camera-on="isScanning = true"
							@camera-off="isScanning = false"
							style="width: 100%; height: 350px; object-fit: cover"
						>
							<!-- Optional: You can put a loading indicator or overlay here -->
							<div v-if="!scanResult" class="scanning-overlay">
								<div class="scan-line"></div>
								<div class="scan-corners">
									<div class="corner top-left"></div>
									<div class="corner top-right"></div>
									<div class="corner bottom-left"></div>
									<div class="corner bottom-right"></div>
								</div>
							</div>
						</qrcode-stream>
					</div>

					<!-- Status messages -->
					<div class="status-messages pa-3">
						<v-alert v-if="errorMessage" type="error" variant="tonal" class="mb-2">
							<v-icon>mdi-alert-circle</v-icon>
							{{ errorMessage }}
						</v-alert>

						<v-alert v-if="scanResult" type="success" variant="tonal" class="mb-2">
							{{ __("Successfully scanned:") }} <strong>{{ scanResult }}</strong> <br /><small
								>Format: {{ scanFormat }}</small
							>
						</v-alert>

						<v-alert
							v-if="!scanResult && !errorMessage && isScanning && scannerDialog"
							type="info"
							variant="tonal"
						>
							{{ __("Position the QR code or barcode within the scanning area") }}
							<br /><small>{{ __("Detecting formats:") }} {{ readerFormats.join(", ") }}</small>
						</v-alert>
					</div>
				</div>

				<!-- Camera permission denied message -->
				<div v-else class="pa-4 text-center">
					<v-icon size="64" color="error">mdi-camera-off</v-icon>
					<h3 class="mt-2">{{ __("Camera Access Required") }}</h3>
					<p class="mt-2">{{ __("Please allow camera access to scan codes") }}</p>
					<!-- Requesting permission is handled by the browser when QrcodeStream tries to access camera -->
				</div>
			</v-card-text>

			<!-- Action buttons -->
			<v-card-actions class="justify-space-between pa-3">
				<div class="d-flex gap-2">
					<!-- Flashlight toggle -->
					<v-btn
						v-if="isScanning && cameras.length > 0"
						@click="toggleTorch"
						:color="torchActive ? 'warning' : 'default'"
						variant="outlined"
						size="small"
					>
						<v-icon>{{ torchActive ? "mdi-flashlight" : "mdi-flashlight-off" }}</v-icon>
						{{ torchActive ? __("Flash On") : __("Flash Off") }}
					</v-btn>

					<!-- Camera switch -->
					<v-btn
						v-if="isScanning && cameras.length > 1"
						@click="switchCamera"
						color="default"
						variant="outlined"
						size="small"
					>
						<v-icon>mdi-camera-switch</v-icon>
						{{ __("Switch Camera") }}
					</v-btn>
				</div>

				<!-- Cancel button -->
				<v-btn @click.stop="stopScanning" color="error" variant="outlined">
					{{ __("Cancel") }}
				</v-btn>
			</v-card-actions>
		</v-card>
	</v-dialog>
</template>

<style scoped>
.scanner-container {
	position: relative;
	overflow: hidden;
	background: #121212;
}

.barcode-scanner {
	position: relative;
}

.scanning-overlay {
	position: absolute;
	top: 0;
	left: 0;
	right: 0;
	bottom: 0;
	pointer-events: none;
	z-index: 10;
}

.scan-line {
	position: absolute;
	top: 50%;
	left: 10%;
	right: 10%;
	height: 2px;
	background: linear-gradient(90deg, transparent, #4caf50, transparent);
	animation: scan 2s linear infinite;
}

@keyframes scan {
	0% {
		transform: translateY(-100px);
	}

	100% {
		transform: translateY(100px);
	}
}

.scan-corners {
	position: absolute;
	top: 20%;
	left: 20%;
	right: 20%;
	bottom: 20%;
}

.corner {
	position: absolute;
	width: 20px;
	height: 20px;
	border: 3px solid #4caf50;
}

.corner.top-left {
	top: 0;
	left: 0;
	border-right: none;
	border-bottom: none;
}

.corner.top-right {
	top: 0;
	right: 0;
	border-left: none;
	border-bottom: none;
}

.corner.bottom-left {
	bottom: 0;
	left: 0;
	border-right: none;
	border-top: none;
}

.corner.bottom-right {
	bottom: 0;
	right: 0;
	border-left: none;
	border-top: none;
}

.status-messages {
	background: rgba(255, 255, 255, 0.95);
}
</style>

<script>
import { QrcodeStream } from "vue-qrcode-reader";

export default {
	name: "CameraScanner",
	components: { QrcodeStream },
	props: {
		scanType: { type: String, default: "Both" }, // 'QR Code', 'Barcode', 'Both'
		closeOnScan: { type: Boolean, default: true },
		resumeAfterScan: { type: Boolean, default: false },
		resumeDelay: { type: Number, default: 1000 },
	},
	data() {
		return {
			scannerDialog: false,
			scanResult: "",
			scanFormat: "",
			errorMessage: "",
			cameraPermissionDenied: false,
			isScanning: false,
			torchActive: false,
			selectedDeviceId: null, // For camera switching
			cameras: [], // Available cameras
		};
	},

	computed: {
		readerFormats() {
			// Define the formats based on scanType prop or default to all common ones
			// Ensure these format names are valid as per vue-qrcode-reader documentation
			const availableFormats = [
				"qr_code",
				"ean_13",
				"ean_8",
				"code_128",
				"code_39",
				"code_93",
				"codabar",
				"upc_a",
				"upc_e",
				"itf", // ITF (Interleaved 2 of 5)
				// Add other formats if needed and supported by zxing-wasm
			];

			if (this.scanType === "QR Code") {
				return ["qr_code"];
			}
			if (this.scanType === "Barcode") {
				return availableFormats.filter((f) => f !== "qr_code");
			}
			return availableFormats; // 'Both'
		},
	},

	methods: {
		async startScanning() {
			this.scannerDialog = true;
			this.errorMessage = "";
			this.scanResult = "";
			this.scanFormat = "";
			this.cameraPermissionDenied = false;
			this.isScanning = true; // QrcodeStream will attempt to start camera automatically
			// We might need to await this.$nextTick() if QrcodeStream is inside v-if controlled by scannerDialog
			await this.$nextTick();
			// Camera listing can be done here or in a dedicated method
			// vue-qrcode-reader doesn't directly list cameras in QrcodeStream,
			// but we can use navigator.mediaDevices.enumerateDevices()
			await this.listCameras();
		},

		async listCameras() {
			try {
				if (!navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices) {
					console.warn("MediaDevices API not supported.");
					this.cameras = [];
					return;
				}
				const devices = await navigator.mediaDevices.enumerateDevices();
				this.cameras = devices.filter((device) => device.kind === "videoinput");
				if (this.cameras.length > 0 && !this.selectedDeviceId) {
					// Select the first available camera by default, or environment facing if possible
					const rearCamera = this.cameras.find((camera) =>
						/back|rear|environment/i.test(camera.label),
					);
					this.selectedDeviceId = rearCamera ? rearCamera.deviceId : this.cameras[0].deviceId;
				}
			} catch (error) {
				console.error("Error listing cameras:", error);
				this.cameras = [];
			}
		},

		onDetect(detectedCodes) {
			// detectedCodes is an array of objects, each with rawValue, format, etc.
			if (detectedCodes && detectedCodes.length > 0) {
				const firstResult = detectedCodes[0];
				this.scanResult = firstResult.rawValue;
				this.scanFormat = firstResult.format;
				this.errorMessage = "";

				this.$emit("barcode-scanned", this.scanResult);

				if (typeof frappe !== "undefined" && frappe.show_alert) {
					frappe.show_alert(
						{
							message: this.__("Code scanned successfully") + ` (${this.scanFormat})`,
							indicator: "green",
						},
						3,
					);
				}

				// Single-shot by default: close dialog and release camera to avoid UI freeze
				if (this.closeOnScan) {
					this.stopScanning();
					return;
				}

				// Continuous mode: temporarily pause then resume
				if (this.resumeAfterScan) {
					this.isScanning = false;
					setTimeout(() => {
						this.scanResult = "";
						this.scanFormat = "";
						this.isScanning = true;
					}, this.resumeDelay || 1000);
				}
			}
		},

		onError(error) {
			this.errorMessage = error.name || "Unknown error";
			if (error.name === "NotAllowedError") {
				this.cameraPermissionDenied = true;
				this.errorMessage = this.__(
					"Camera permission denied. Please allow camera access in your browser settings.",
				);
			} else if (error.name === "NotFoundError" || error.name === "DevicesNotFoundError") {
				this.errorMessage = this.__("No camera found on this device.");
			} else if (error.name === "NotSupportedError") {
				this.errorMessage = this.__("Secure context (HTTPS) required for camera access.");
			} else if (error.name === "AbortError") {
				this.errorMessage = this.__("Camera access aborted.");
			} else {
				this.errorMessage = this.__("Error accessing camera:") + ` ${error.message}`;
			}
			console.error("Camera error:", error);
			this.isScanning = false;
		},

		stopScanning() {
			// Update state first so QrcodeStream tears down
			this.isScanning = false;
			this.scannerDialog = false;
			this.torchActive = false;

			// Attempt to stop and release any active media tracks to avoid freezes
			try {
				const root = this.$el;
				if (root) {
					const videos = root.querySelectorAll("video");
					videos.forEach((v) => {
						try {
							if (typeof v.pause === "function") v.pause();
							const stream = v.srcObject;
							if (stream && typeof stream.getTracks === "function") {
								stream.getTracks().forEach((t) => {
									try {
										t.stop();
									} catch (_) {}
								});
							}
							// Clear srcObject/src to prevent "Invalid URI" warnings
							v.srcObject = null;
							v.removeAttribute("src");
							if (typeof v.load === "function") v.load();
						} catch (e) {
							console.debug("Video cleanup failed", e);
						}
					});
				}
			} catch (err) {
				console.debug("Camera cleanup error", err);
			}

			// Reset feedback fields
			this.scanResult = "";
			this.scanFormat = "";
			this.errorMessage = "";

			// selectedDeviceId and cameras can remain as they are for next scan
			this.$emit("scanner-closed");
		},

		async toggleTorch() {
			this.torchActive = !this.torchActive;
			// The QrcodeStream component has a :torch prop, binding this.torchActive to it should work.
		},

		async switchCamera() {
			if (this.cameras.length > 1) {
				const currentIndex = this.cameras.findIndex((cam) => cam.deviceId === this.selectedDeviceId);
				const nextIndex = (currentIndex + 1) % this.cameras.length;
				this.selectedDeviceId = this.cameras[nextIndex].deviceId;
				// QrcodeStream should react to changes in :camera prop (if we bind selectedDeviceId to it)
				// Or we might need to re-initialize or force a re-render of QrcodeStream
				// Forcing re-render by toggling isScanning or using a v-if with a key
				this.isScanning = false;
				await this.$nextTick();
				this.isScanning = true;
				if (typeof frappe !== "undefined" && frappe.show_alert) {
					frappe.show_alert(
						{
							message:
								this.__("Switched to: ") +
								(this.cameras[nextIndex].label || `Camera ${nextIndex + 1}`),
							indicator: "blue",
						},
						2,
					);
				}
			}
		},

		handleEscKey(event) {
			if (event.key === "Escape" && this.scannerDialog) {
				event.preventDefault();
				this.stopScanning();
			}
		},
	},

	watch: {
		scannerDialog(newVal) {
			if (newVal) {
				// When dialog opens, if no camera is selected, list them.
				if (!this.selectedDeviceId && this.cameras.length === 0) {
					this.listCameras();
				}
			} else {
				// When dialog closes, ensure scanning is fully stopped.
				this.stopScanning();
			}
		},
	},

	mounted() {
		if (typeof document !== "undefined") {
			document.addEventListener("keydown", this.handleEscKey);
		}
	},

	beforeUnmount() {
		if (typeof document !== "undefined") {
			document.removeEventListener("keydown", this.handleEscKey);
		}
		this.stopScanning(); // Ensure scanner stops when component is unmounted
	},
};
</script>
