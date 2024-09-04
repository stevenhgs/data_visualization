<script>
// @ts-nocheck

	import { onMount } from 'svelte';
	import { json, geoTransform, geoPath } from 'd3';
	import * as htmlVars from './htmlVars';
	
	// data access
	export let data = [];

	// code for world map
	let width = 100;
	let height = 100;
	$: usedWidth = 900;
	$: usedHeight = height - 50;
	$: scaleFactor = 0.75;
	const minScaleFactor = 0.75;
	const maxScaleFactor = 7;
	let scaleFactorIncrement = 0.5;
	$: translateX = 100;
	$: translateY = 50;

	$: originTranslateX = 0;
	$: originTranslateY = 0;
	$: originXMouseDown = 0;
	$: originYMouseDown = 0;

	// filter data
	// plant filtering
	$: selectedPlantsMask = 31;
	selectedPlantsMask = 31;

	// product filtering
	$: selectedProductsMask = 3;
	selectedProductsMask = 3;

	// type of data selector
	$: selectedTypeOfDataText = "sales";
	selectedTypeOfDataText = "sales";


	// radius selection for sales
	$: salesMinMaxSelector = "totalSales";
	salesMinMaxSelector = "totalSales";

	// period filtering
	$: minMonth = 1;
	$: minYear = 2022;
	$: maxMonth = 12;
	$: maxYear = 2024;


	// Variables for data analysis text
	$: numberOfCustomers = 0;
	$: numberOfCities = 0;
	$: numberOfCarBatteries = 0;
	$: numberOfHomeBatteries = 0;
	$: numberOfOrders = 0;
	$: numberOfDelays = 0;

	let geojsonPath = "http://localhost:5173/europe.geojson"
	let geojson;
	json(geojsonPath).then((data) => geojson = data);

	const minLon = -24.542225;
	const maxLon = 50.37499;
	const minLat = 29.486706;
	const maxLat = 71.154709;

	const minRadius = 10;
	$: maxRadius = 60;
	maxRadius = 60;

	// histogram variables
	const numberOfBins = 30;
	const maxHeightOfBin = 165;
	const histogramWidth = 300;
	const barWidth = histogramWidth / numberOfBins;
	$: binValueForHistogram = Array(numberOfBins).fill(0);
	$: binRangesForHistogram = Array(numberOfBins + 1).fill(0);
	$: maxOccurenceForHistogram = 1;
	maxOccurenceForHistogram = 1;


	$: rescaleHeightOfBin = function(value) {
		let rangeMin = 0;
		let rangeMax = maxHeightOfBin;
		return (((rangeMax - rangeMin)*(value-0))/(maxOccurenceForHistogram-0)) + rangeMin;
	}

	let getValuesOfBins = function(dataMap) {
		let binRanges = Array(numberOfBins).fill(0);
		let binValues = Array(numberOfBins).fill(0);

		const increments = maxValueForRadius / numberOfBins;
		binRanges[0] = increments;
		for (let i = 1; i < numberOfBins - 1; i++) {
			binRanges[i] = binRanges[i-1] + increments
		}
		binRanges[numberOfBins-1] = maxValueForRadius;

		for (const dataMapKeys of Array.from(dataMap.keys())) {
			let value = dataMap.get(dataMapKeys).get(salesMinMaxSelector);
			for (let i = 0; i < numberOfBins; i++) {
				if (value <= binRanges[i]) {
					binValues[i] += 1;
					break;
				}
			}
		}
		return [binValues, binRanges];
	}
	
	$: rescale_lon = function(longitude, givenWidth) {
		let range_min = 0;
		let range_max = givenWidth;
      	return (((range_max - range_min)*(longitude-minLon))/(maxLon-minLon)) + range_min;
  	}

	$: rescale_lat = function(latitude, givenHeight) {
		let range_min = givenHeight;
		let range_max = 0;
      	return (((range_max - range_min)*(latitude-minLat))/(maxLat-minLat)) + range_min;
  	}

	$: projection = geoTransform({
      point: function(x, y) {
          this.stream.point(rescale_lon(x, usedWidth), rescale_lat(y, usedHeight));
      }
    });
	$: pathGenerator = geoPath(projection);

	let countries = []
	$: if (geojson) countries = geojson.features.map(feature => {
		return {
			...feature,
			path: pathGenerator(feature)
		};
	});

	// code to get coordinates from city + country
	let address_to_coordinates_map = new Map();
	for (let i = 0; i < data.coordinates.length; i++) {
		address_to_coordinates_map.set(data.coordinates[i].address, [data.coordinates[i].longitude, data.coordinates[i].latitude]);
	}

	// code to get country to plant map
	let countryToPlantKeyMap = new Map();
	for (let i = 0; i < data.customerPlant.length; i++) {
		countryToPlantKeyMap.set(data.customerPlant[i].CustomerCountry, data.customerPlant[i].PlantKey);
	}

	// code to get coordinates from customerKey
	let customerKeyToLonLat = new Map();
	let customerKeyToCountry = new Map();
	for (let i = 0; i < data.customers.length; i++) {
		let lonLat = address_to_coordinates_map.get(data.customers[i].CustomerCity + " " + data.customers[i].CustomerCountry);
		customerKeyToLonLat.set(data.customers[i].CustomerKey, lonLat);
		customerKeyToCountry.set(data.customers[i].CustomerKey, data.customers[i].CustomerCountry);
	}

	// code to get city from coordinates
	let lonLatToCity = new Map();
	for (let i = 0; i < data.customers.length; i++) {
		let lonLat = address_to_coordinates_map.get(data.customers[i].CustomerCity + " " + data.customers[i].CustomerCountry);
		lonLatToCity.set(lonLat, data.customers[i].CustomerCity);
	}

	$: changeScaleFactor = function(increase) {
		scaleFactor = Math.min(maxScaleFactor, Math.max(minScaleFactor, scaleFactor + increase));
	}

	$: rescaleRadius = function(value) {
		let range_min = minRadius;
		let range_max = maxRadius;
      	return ((range_max - range_min)*(value-minValueForRadius))/(maxValueForRadius-minValueForRadius) + range_min
  	}

	function onMouseMove (event) {
		translateX = originTranslateX + ((event.clientX - originXMouseDown) / scaleFactor);
		translateY = originTranslateY + ((event.clientY - originYMouseDown) / scaleFactor);
  	}

	function onMouseDown (event) {
		originTranslateX = translateX;
		originTranslateY = translateY;
		originXMouseDown = event.clientX;
		originYMouseDown = event.clientY;
		addEventListener('mousemove', onMouseMove);
		addEventListener('mouseup', onMouseUp);
	}

	function onMouseUp () {
		removeEventListener('mousemove', onMouseMove);
		removeEventListener('mouseup', onMouseUp);
	}

	function getDelayDataMap() {
		/**
		 * delayDataMap: Map[
		 * 					[lon,lat]: Map[
		 * 									"country": country,
		 * 									"nbOrders": int,
		 * 									"daysOfDelay": int,
		 * 								    ClientName: Map[
		 * 														"nbOrders": int
		 * 														"daysOfDelay": int
		 * 													]
		 *								  ]
		 * 				   ]
		*/
		let delayDataMap = new Map();
		let nbCities = 0;
		let nbCustomers = 0;
		let nbOfDelays = 0;
		let nbOfOrders = 0;
		
		for (let i = 0; i < data.salesWithDelays.length; i++) {
			let year = Number(data.salesWithDelays[i].SalesOrderCreationDate.slice(0, 4));
			let month = Number(data.salesWithDelays[i].SalesOrderCreationDate.slice(5, 7));
			let plantKey = Number(data.salesWithDelays[i].PlantKey);
			let plantFlag = 1 << (plantKey - 4);
			let productKey = Number(data.salesWithDelays[i].MaterialKey);
			let productFlag = 1 << (productKey - 1);

			if (year < minYear || year > maxYear || (year == minYear && month < minMonth) || (year == maxYear && month > maxMonth)) {
				continue;
			}
			if ((plantFlag & selectedPlantsMask) === 0) {
				continue;
			}
			if ((productFlag & selectedProductsMask) === 0) {
				continue;
			}
			let latLon = customerKeyToLonLat.get(data.salesWithDelays[i].CustomerKey);
			let delay = Number(data.salesWithDelays[i].delay);
			nbOfOrders += 1;
			if (delay < 0) { 
				delay = 0; 
			} else {
				nbOfDelays += 1;
			}
			if (delayDataMap.has(latLon)) {
				let daysOfDelay = delayDataMap.get(latLon).get("daysOfDelay");
				let nbOrders = delayDataMap.get(latLon).get("nbOrders");
				delayDataMap.get(latLon).set("daysOfDelay", daysOfDelay + delay);
				delayDataMap.get(latLon).set("nbOrders", nbOrders + 1);
				if (delayDataMap.get(latLon).has(data.salesWithDelays[i].CustomerKey)) {
					let customerDaysOfDelay = delayDataMap.get(latLon).get(data.salesWithDelays[i].CustomerKey).get("daysOfDelay");
					let customerNbOrders = delayDataMap.get(latLon).get(data.salesWithDelays[i].CustomerKey).get("nbOrders");
					delayDataMap.get(latLon).get(data.salesWithDelays[i].CustomerKey).set("daysOfDelay", customerDaysOfDelay + delay);
					delayDataMap.get(latLon).get(data.salesWithDelays[i].CustomerKey).set("nbOrders", customerNbOrders + 1);
				} else {
					nbCustomers += 1;
					delayDataMap.get(latLon).set(data.salesWithDelays[i].CustomerKey, new Map([["daysOfDelay", delay], ["nbOrders", 1]]));
				}
			} else {
				nbCities += 1;
				nbCustomers += 1;
				delayDataMap.set(latLon, new Map());
				delayDataMap.get(latLon).set("country", customerKeyToCountry.get(data.salesWithDelays[i].CustomerKey));
				delayDataMap.get(latLon).set("nbOrders", 1);
				delayDataMap.get(latLon).set("daysOfDelay", delay);
				delayDataMap.get(latLon).set(data.salesWithDelays[i].CustomerKey, new Map([["daysOfDelay", delay], ["nbOrders", 1]]));
			}
		}

		let bigNum = 1000000000000000;
		let minQuantity = bigNum;
		let maxQuantity = 0;
		for (const lonLatKey of Array.from(delayDataMap.keys())) {
			let avgDelay = delayDataMap.get(lonLatKey).get("daysOfDelay") / delayDataMap.get(lonLatKey).get("nbOrders");
			if (avgDelay === 0) {
				delayDataMap.delete(lonLatKey);
			} else {
				delayDataMap.get(lonLatKey).set("averageDelay", avgDelay);
				minQuantity = Math.min(minQuantity, avgDelay);
				maxQuantity = Math.max(maxQuantity, avgDelay);
			}
		}
		if (minQuantity == bigNum) { minQuantity = 0; }

		let extraDataMap = new Map([["minQuantity", minQuantity], ["maxQuantity", maxQuantity], 
									["nbCustomers", nbCustomers], ["nbCities", nbCities], ["nbOfOrders", nbOfOrders],
									["nbOfDelays", nbOfDelays]])
		return [delayDataMap, extraDataMap];

	}

	function getSaleDataMap() {
		/**
		 * saleDataMap: Map[
		 * 					[lon,lat]: Map[
		 * 									"country": str,
		 * 									"totalSales": int,
		 * 									"maxSales": int,
		 * 								    ClientName: int
		 *								  ]
		 * 				   ]
		*/
		let saleDataMap = new Map();
		let nbCities = 0;
		let nbCustomers = 0;
		let nbCarBatteries = 0;
		let nbHomeBatteries = 0;
		let maxQuantity = 0;
		for (let i = 0; i < data.sales.length; i++) {
			let year = Number(data.sales[i].SalesOrderCreationDate.slice(0, 4));
			let month = Number(data.sales[i].SalesOrderCreationDate.slice(5, 7));
			let plantKey = Number(data.sales[i].PlantKey);
			let plantFlag = 1 << (plantKey - 4);
			let productKey = Number(data.sales[i].MaterialKey);
			let productFlag = 1 << (productKey - 1);

			if (year < minYear || year > maxYear || (year == minYear && month < minMonth) || (year == maxYear && month > maxMonth)) {
				continue;
			}
			if ((plantFlag & selectedPlantsMask) === 0) {
				continue;
			}
			if ((productFlag & selectedProductsMask) === 0) {
				continue;
			}
			if (productKey == 1) {
				nbCarBatteries += Number(data.sales[i].OrderQuantity);
			} else {
				nbHomeBatteries += Number(data.sales[i].OrderQuantity);
			}
			let latLon = customerKeyToLonLat.get(data.sales[i].CustomerKey);
			let quantity = Number(data.sales[i].OrderQuantity);
			if (saleDataMap.has(latLon)) {
				let salesQuantity = saleDataMap.get(latLon).get("totalSales");
				saleDataMap.get(latLon).set("totalSales", salesQuantity + quantity);
				if (saleDataMap.get(latLon).has(data.sales[i].CustomerKey)) {
					let customerSalesQuantity = saleDataMap.get(latLon).get(data.sales[i].CustomerKey);
					saleDataMap.get(latLon).set(data.sales[i].CustomerKey, customerSalesQuantity + quantity);
				} else {
					nbCustomers += 1;
					saleDataMap.get(latLon).set(data.sales[i].CustomerKey, quantity);
				}
				let maxSalesQuantity = saleDataMap.get(latLon).get("maxSales");
				let potentialNewMaxSalesQuantity = saleDataMap.get(latLon).get(data.sales[i].CustomerKey);
				saleDataMap.get(latLon).set("maxSales", Math.max(maxSalesQuantity, potentialNewMaxSalesQuantity));
			} else {
				nbCities += 1;
				nbCustomers += 1;
				saleDataMap.set(latLon, new Map());
				saleDataMap.get(latLon).set("country", customerKeyToCountry.get(data.sales[i].CustomerKey));
				saleDataMap.get(latLon).set("totalSales", quantity);
				saleDataMap.get(latLon).set("maxSales", quantity);
				saleDataMap.get(latLon).set(data.sales[i].CustomerKey, quantity);
			}
			maxQuantity = Math.max(maxQuantity, saleDataMap.get(latLon).get(salesMinMaxSelector));
		}

		let bigNum = 1000000000000000;
		let minQuantity = bigNum;
		for (const lonLatKey of Array.from(saleDataMap.keys())) {
			minQuantity = Math.min(minQuantity, saleDataMap.get(lonLatKey).get(salesMinMaxSelector));
		}
		if (minQuantity == bigNum) { minQuantity = 0; }

		let extraDataMap = new Map([["minQuantity", minQuantity], ["maxQuantity", maxQuantity], 
									["nbCustomers", nbCustomers], ["nbCities", nbCities], ["nbCarBatteries", nbCarBatteries],
									["nbHomeBatteries", nbHomeBatteries]])
		return [saleDataMap, extraDataMap];
	}

	let dataMapReturn = getSaleDataMap();
	$: filteredDataMap = dataMapReturn[0];
	$: filteredDataMapKeys = Array.from(filteredDataMap.keys());
	$: minValueForRadius = dataMapReturn[1].get("minQuantity");
	$: maxValueForRadius = dataMapReturn[1].get("maxQuantity");


	function refreshVariablesAfterFilterChange() {
		let dataMapReturn;
		if (selectedTypeOfDataText == "sales") {
			dataMapReturn = getSaleDataMap();
			numberOfCarBatteries = dataMapReturn[1].get("nbCarBatteries");
			numberOfHomeBatteries = dataMapReturn[1].get("nbHomeBatteries");
		} else {
			dataMapReturn = getDelayDataMap();
			numberOfOrders = dataMapReturn[1].get("nbOfOrders");
			numberOfDelays = dataMapReturn[1].get("nbOfDelays");
		}
		filteredDataMap = dataMapReturn[0];
		filteredDataMapKeys = Array.from(filteredDataMap.keys());
		minValueForRadius = dataMapReturn[1].get("minQuantity");
		maxValueForRadius = dataMapReturn[1].get("maxQuantity");
		numberOfCustomers = dataMapReturn[1].get("nbCustomers");
		numberOfCities = dataMapReturn[1].get("nbCities");
		
		// histogram
		let HistogramData = getValuesOfBins(filteredDataMap);
		binValueForHistogram = HistogramData[0];
		binRangesForHistogram = [0].concat(HistogramData[1]);
		maxOccurenceForHistogram = Math.max(...binValueForHistogram);

		// set text
		setAnalysisText();
	}

	const monthIndexToMonthMap = new Map([[1, "January"], [2, "February"], [3, "March"], [4, "April"],
											[5, "May"], [6, "June"], [7, "July"], [8, "August"],
											[9, "September"], [10, "Octobre"], [11, "November"], [12, "December"],]);

	function setAnalysisText() {
		// period text
		let periodText = document.getElementById("periodText");
		periodText.innerHTML = `The shown data is from the period between <strong>${monthIndexToMonthMap.get(minMonth)} ${minYear}</strong> until <strong>${monthIndexToMonthMap.get(maxMonth)} ${maxYear}</strong>.`;
		// customer text
		let customerText = document.getElementById("customerText");
		customerText.innerHTML = `There are <strong>${numberOfCustomers} different customers</strong> in <strong>${numberOfCities} different cities</strong>.`;
		// product text
		let productText = document.getElementById("productText");
		let smallRadiusText = document.getElementById("small-radius-text");
		let bigRadiusText = document.getElementById("big-radius-text");
		// histogram text
		let valueOfHistogramDescriptionText = document.getElementById("valueOfHistogramDescription");
		let maxValueHistogramText = document.getElementById("maxValueHistogram");
		let maxYValueHistogramText = document.getElementById("maxYValueHistogram");
		let histogramDescriptionText = document.getElementById("histogramDescription");


		if (selectedTypeOfDataText == "sales") {
			if (selectedProductsMask == 3) {
				productText.innerHTML = `There were <strong>${(numberOfCarBatteries).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")} car batteries</strong> and <strong>${(numberOfHomeBatteries).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")} home batteries</strong> sold.`;
			} else if (selectedProductsMask == 1) {
				productText.innerHTML = `There were <strong>${(numberOfCarBatteries).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")} car batteries</strong> sold.`;
			} else if (selectedProductsMask == 2) {
				productText.innerHTML = `There were <strong>${(numberOfHomeBatteries).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")} home batteries</strong> sold.`;
			} else {
				productText.innerHTML = `There are <strong>no</strong> products selected`;
			}
			// radius text
			smallRadiusText.innerHTML = `<strong>${(minValueForRadius).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}</strong> ${selectedTypeOfDataText}`;
			bigRadiusText.innerHTML = `<strong>${(maxValueForRadius).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}</strong> ${selectedTypeOfDataText}`;
			maxValueHistogramText.innerHTML = `<strong>${(maxValueForRadius).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}</strong>`;
		} else {
			productText.innerHTML = `There were <strong>${(numberOfOrders).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")} orders</strong> of which <strong>${(numberOfDelays).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}</strong> were <strong>delayed</strong>.`;
			smallRadiusText.innerHTML = `<strong>${(minValueForRadius).toFixed(2)}</strong> ${selectedTypeOfDataText}`;
			bigRadiusText.innerHTML = `<strong>${(maxValueForRadius).toFixed(2)}</strong> ${selectedTypeOfDataText}`;
			maxValueHistogramText.innerHTML = `<strong>${(maxValueForRadius).toFixed(2)}</strong>`;
		}
		valueOfHistogramDescriptionText.innerHTML = selectedTypeOfDataText;
		maxYValueHistogramText.innerHTML = `<strong>${maxOccurenceForHistogram}</strong>`;
		histogramDescriptionText.innerHTML = `Cities ${selectedTypeOfDataText} histogram`;
	}

	onMount(() => {
        // This code now runs only on the client after the component is mounted
        const rangeInput = document.querySelectorAll(".range-input input");
		let progress = document.querySelector(".slider .progress");
		let fromRangeTextElement = document.querySelector(".from-range-text");
		let untilRangeTextElement = document.querySelector(".until-range-text");
		let plantCheckBoxes = document.querySelectorAll("#plant-selector input[type='checkbox']");
		let productCheckBoxes = document.querySelectorAll("#product-selector input[type='checkbox']");
		let radiusSelectorCheckBoxes = document.querySelectorAll("#radius-selector input[type='checkbox']");
		let typeOfDataSelectorCheckBoxes = document.querySelectorAll("#type-of-data-selector input[type='checkbox']");

		// set initial analysis text
		refreshVariablesAfterFilterChange();
		setAnalysisText();
		

		// code for period range
        rangeInput.forEach(input => {
            input.addEventListener("input", () => {
				rangeInput[1].value = Math.max(rangeInput[0].value, rangeInput[1].value);
				rangeInput[0].value = Math.min(rangeInput[0].value, rangeInput[1].value);
				

                let minVal = parseInt(rangeInput[0].value);
                let maxVal = parseInt(rangeInput[1].value);

				let leftPercentage = ((minVal / rangeInput[0].max) * 100) + "%";
				let rightPercentage = 100 - ((maxVal / rangeInput[0].max) * 100) + "%";

				minMonth = (minVal % 12) + 1;
				minYear = Math.floor(minVal / 12) + 2022;
				maxMonth = (maxVal % 12) + 1;
				maxYear = Math.floor(maxVal / 12) + 2022;

				fromRangeTextElement.innerHTML = `From: ${minMonth}-${minYear}`;
				untilRangeTextElement.innerHTML = `Until: ${maxMonth}-${maxYear}`;

				progress.style.left = leftPercentage;
				progress.style.right = rightPercentage;

				refreshVariablesAfterFilterChange();
            });
        });

		// code for plant selector
		plantCheckBoxes.forEach(input => {
            input.addEventListener("change", () => {
				let newSelectedPlantsMask = 0;
				for (let i = 0; i < plantCheckBoxes.length; i++) {
					if (plantCheckBoxes[i].checked) {
						newSelectedPlantsMask |= 1 << i
					}
				}
				selectedPlantsMask = newSelectedPlantsMask
				refreshVariablesAfterFilterChange();
			});
		});

		// code for product selector
		productCheckBoxes.forEach(input => {
            input.addEventListener("change", () => {
				let newSelectedProductsMask = 0;
				for (let i = 0; i < 2; i++) {
					if (productCheckBoxes[i].checked) {
						newSelectedProductsMask |= 1 << i
					}
				}
				selectedProductsMask = newSelectedProductsMask
				refreshVariablesAfterFilterChange();
			});
		});

		// code for radius selector
		typeOfDataSelectorCheckBoxes.forEach(input => {
            input.addEventListener("change", () => {
				if (selectedTypeOfDataText == "sales") {
					if (!typeOfDataSelectorCheckBoxes[1].checked) {
						typeOfDataSelectorCheckBoxes[0].checked = true;
					} else {
						typeOfDataSelectorCheckBoxes[0].checked = false;
						selectedTypeOfDataText = "avg. delay (days)";
						salesMinMaxSelector = "averageDelay";
						maxRadius = 30;
						// make sales related checkboxes unclickable
						for (const radiusSelectorCheckBox of radiusSelectorCheckBoxes) {
							radiusSelectorCheckBox.disabled = true;
						}
					}
				} else {
					if (!typeOfDataSelectorCheckBoxes[0].checked) {
						typeOfDataSelectorCheckBoxes[1].checked = true;
					} else {
						typeOfDataSelectorCheckBoxes[1].checked = false;
						selectedTypeOfDataText = "sales";
						if (radiusSelectorCheckBoxes[0].checked) {
							salesMinMaxSelector = "totalSales";
							maxRadius = 60;
						} else {
							salesMinMaxSelector = "maxSales";
							maxRadius = 30;
						}
						// make sales related checkboxes clickable
						for (const radiusSelectorCheckBox of radiusSelectorCheckBoxes) {
							radiusSelectorCheckBox.disabled = false;
						}
					}
				}
				refreshVariablesAfterFilterChange(selectedTypeOfDataText);
			});
		});

		// code for radius selector
		radiusSelectorCheckBoxes.forEach(input => {
            input.addEventListener("change", () => {
				if (salesMinMaxSelector == "totalSales") {
					if (!radiusSelectorCheckBoxes[1].checked) {
						radiusSelectorCheckBoxes[0].checked = true;
					} else {
						radiusSelectorCheckBoxes[0].checked = false;
						salesMinMaxSelector = "maxSales";
						maxRadius = 30;
					}
				} else {
					if (!radiusSelectorCheckBoxes[0].checked) {
						radiusSelectorCheckBoxes[1].checked = true;
					} else {
						radiusSelectorCheckBoxes[1].checked = false;
						salesMinMaxSelector = "totalSales";
						maxRadius = 60;
					}
				}
				refreshVariablesAfterFilterChange();
			});
		});
	})

	// hover
	$: selectedLatLonKey = undefined;
	$: selectedPlant = undefined;
	$: selectedHistogramIndex = undefined;
	let mouse_x, mouse_y, scroll_y;
	const setMousePosition = function(event) {
		mouse_x = event.clientX;
		mouse_y = event.clientY;
		scroll_y = window.scrollY;
	}

</script>


<!-- Components -->
<!-- Load the custome style -->
<svelte:head>
    <link rel="stylesheet" href="./src/routes/page.css">
</svelte:head>

<div class="filter-div" style={`border:1px solid black; position: absolute; left: 5px; width: ${usedWidth}px; height: 220px; overflow: hidden;`}>
	<div class="filter-sub-div-0" id="Header" style="height: 40px; width: 100%; top: 0px; border-bottom: 1px solid black; box-sizing: border-box;">
		<h2 class="sub-title" style="position: absolute; top: -18px; left: 10px">Data Filtering</h2>
	</div>
	
	<div class="filter-sub-div-1" id="plant-selector" style="height: 30px; width: 100%; top: 40px">
		<div class="text" style={`right: ${htmlVars.startXPositionForFilteringTextFromRight}px; top: ${htmlVars.percentageForDivText}%`}>Plants: </div>
		<input type="checkbox" class="plant" id="Plant4" checked style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 0}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 0}px; top: ${htmlVars.percentageForDivText}%`}>Antwerp</div>
		<input type="checkbox" class="plant" id="Plant5" checked style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 1}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 1}px; top: ${htmlVars.percentageForDivText}%`}>Wrocław</div>
		<input type="checkbox" class="plant" id="Plant6" checked style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 2}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 2}px; top: ${htmlVars.percentageForDivText}%`}>Lyon</div>
		<input type="checkbox" class="plant" id="Plant7" checked style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 3}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 3}px; top: ${htmlVars.percentageForDivText}%`}>Birmingham</div>
		<input type="checkbox" class="plant" id="Plant8" checked style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 4}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 4}px; top: ${htmlVars.percentageForDivText}%`}>Göteborg</div>
	</div>

	
	<div class="filter-sub-div-2" id="product-selector" style="height: 30px; width: 100%; top: 70px">
		<div class="text" style={`right: ${htmlVars.startXPositionForFilteringTextFromRight}px; top: ${htmlVars.percentageForDivText}%`}>Products: </div>
		<input type="checkbox" class="product" id="Product1" checked style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 0}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 0}px; top: ${htmlVars.percentageForDivText}%`}>Car Battery</div>
		<input type="checkbox" class="product" id="Product2" checked style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 1}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 1}px; top: ${htmlVars.percentageForDivText}%`}>Home Battery</div>
	</div>

	<div class="filter-sub-div-1" id="type-of-data-selector" style="height: 30px; width: 100%; top: 100px">
		<div class="text" style={`right: ${htmlVars.startXPositionForFilteringTextFromRight}px; top: ${htmlVars.percentageForDivText}%`}>Data: </div>
		<input type="checkbox" id="Sales" checked	 style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 0}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 0}px; top: ${htmlVars.percentageForDivText}%`}>Sales</div>
		<input type="checkbox" id="Delay" style={`position: absolute; left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 1}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 1}px; top: ${htmlVars.percentageForDivText}%`}>Delays</div>
	</div>


	<div class="filter-sub-div-2" id="period-selector" style="height: 60px; width: 100%; top: 130px">
		<div class="text" style={`right: ${htmlVars.startXPositionForFilteringTextFromRight}px; top: ${htmlVars.percentageForDivText}%`}>Period: </div>
		<div class="slider">
			<div class="progress"></div>
		</div>
		<div class="range-input">
			<input class="period" type="range" min="0" max="35" value="0">
			<input class="period" type="range" min="0" max="35" value="36">
		</div>
		<div class="from-range-text" style={`position: absolute; left: ${htmlVars.startXPositionForFromText}px`}>
			From: 1-2022
		</div>
		<div class="until-range-text" style={`position: absolute; left: ${htmlVars.startXPositionForUntilText}px`}> 
			Until: 12-2024
		</div>
	</div>


	<div class="filter-sub-div-1" id="radius-selector" style="height: 30px; width: 100%; top: 190px">
		<div class="text" style={`right: ${700}px; top: ${htmlVars.percentageForDivText}%;`}>Radius of city based on: </div>
		<input type="checkbox" id="Sales" checked style={`position: absolute; left: ${75 + htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 0}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${75 + htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 0}px; top: ${htmlVars.percentageForDivText}%`}>Sum of the sales to customers in the city</div>
		<input type="checkbox" id="Delay" style={`position: absolute; left: ${75 + htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenPlantCheckboxes * 2.35}px; top: ${htmlVars.percentageForCheckbox}%`}>
		<div class="text" style={`left: ${75 + htmlVars.startXPositionForPlantCheckBoxes + htmlVars.gapBetweenTextboxAndText + htmlVars.gapBetweenPlantCheckboxes * 2.35}px; top: ${htmlVars.percentageForDivText}%`}>Max sales to a customer in the city</div>
	</div>
</div>


<main bind:clientWidth={width} bind:clientHeight={height} style={`position: absolute; left: 5px; top: 230px;`}>
	<div class="map-div" style={`border:1px solid black; position: absolute; width: ${usedWidth}px; height: ${htmlVars.mapDivHeight}px; overflow: hidden; top: 5px`}>
		<div class="filter-sub-div-0" id="Header" style="height: 40px; width: 100%; border-bottom: 1px solid black; box-sizing: border-box;">
			<h2 class="sub-title" style="position: absolute; top: -18px; left: 10px">Map</h2>
			<div id="map settings" style="position: absolute; right: 3%; top: 15%">
				<button on:click={changeScaleFactor(-scaleFactorIncrement)}><strong>-</strong></button>
				<button on:click={changeScaleFactor(scaleFactorIncrement)}><strong>+</strong></button>
			</div>	
		</div>
		
		
		<!-- svelte-ignore a11y-no-static-element-interactions -->
		<svg class="map" width={usedWidth} height={htmlVars.mapHeight} on:mousedown={onMouseDown}>
			<g transform={"scale (" + String(scaleFactor) + ")"}>
			<g transform={"translate(" + String(translateX) + "," + String(translateY) + ")"}>
			
	
			{#each countries as country}
				<path class=country d={country.path}/>
			{/each}
			
			{#each filteredDataMapKeys as latLonKey}
			<path
				class={"c" + countryToPlantKeyMap.get(filteredDataMap.get(latLonKey).get("country"))}
				d={"M " + String(rescale_lon(latLonKey[0], usedWidth)) + " " + String(rescale_lat(latLonKey[1], usedHeight)) + " l 0.0001 0"}
				style={`stroke-width: ${rescaleRadius(filteredDataMap.get(latLonKey).get(salesMinMaxSelector))}`}
			/>
			{/each}
			
			{#each filteredDataMapKeys as latLonKey}
			<!-- svelte-ignore a11y-no-static-element-interactions -->
			<!-- svelte-ignore a11y-mouse-events-have-key-events -->
			<path
				class="defaultCustomer"
				d={"M " + String(rescale_lon(latLonKey[0], usedWidth)) + " " + String(rescale_lat(latLonKey[1], usedHeight)) + " l 0.0001 0"}
				on:mouseover={function(event) {selectedLatLonKey = latLonKey; setMousePosition(event)}}
				on:mouseout={function() {selectedLatLonKey = undefined}}/>
				/>
			{/each}
			
	
			{#each data.plants as plant}
			<!-- svelte-ignore a11y-no-static-element-interactions -->
			<!-- svelte-ignore a11y-mouse-events-have-key-events -->
			<path
			class={"p" + plant.PlantKey}
			d={"M " + String(rescale_lon(address_to_coordinates_map.get(plant.PlantCity + " " + plant.PlantCountry)[0], usedWidth)) + " " + String(rescale_lat(address_to_coordinates_map.get(plant.PlantCity + " " + plant.PlantCountry)[1], usedHeight)) + " l 0.0001 0"}
			on:mouseover={function(event) {selectedPlant = plant; setMousePosition(event)}}
			on:mouseout={function() {selectedPlant = undefined}}/>
			/>
			{/each}
			</g>
			</g>
			
		</svg>
	</div>
</main>

{#if selectedLatLonKey != undefined}
	<div class="mouse-hover-div" style="left: {mouse_x + 10}px; top: {mouse_y - 10 + scroll_y}px">
		Country: {filteredDataMap.get(selectedLatLonKey).get("country")}<br>
		City: {lonLatToCity.get(selectedLatLonKey)	}<br>
		{#if selectedTypeOfDataText == "sales"}
			Amount of customers: {Array.from(filteredDataMap.get(selectedLatLonKey).keys()).length - 3}<br>
			Total Sales in city: {(filteredDataMap.get(selectedLatLonKey).get("totalSales")).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}<br>
			Max Sales in city:  {(filteredDataMap.get(selectedLatLonKey).get("maxSales")).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}
		{:else}
			Amount of customers: {Array.from(filteredDataMap.get(selectedLatLonKey).keys()).length - 4}<br>
			Average delay in days: {(filteredDataMap.get(selectedLatLonKey).get("averageDelay")).toFixed(2)}<br>
			Number of orders: {(filteredDataMap.get(selectedLatLonKey).get("nbOrders")).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}<br>
			Total of delay in days: {(filteredDataMap.get(selectedLatLonKey).get("daysOfDelay")).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",")}
		{/if}
	</div>
{/if}

{#if selectedPlant != undefined}
	<div class="mouse-hover-div" style="left: {mouse_x + 10}px; top: {mouse_y - 10 + scroll_y}px">
		Plant: {selectedPlant.PlantName}
	</div>
{/if}


<div class="analysis-div" style={`border:1px solid black; overflow: hidden; position: absolute; width: 400px; height: 707px; left: 912px;`}>
	<div class="filter-sub-div-0" id="Header" style="height: 40px; width: 100%; top: 0px; border-bottom: 1px solid black; box-sizing: border-box;">
		<h2 class="sub-title" style="position: absolute; top: -18px; left: 10px">Data Details</h2>
	</div>

	<div class="analysis-sub-div-1" id="periodTextDiv" style="height: 90px; width: 100%; top: 40px">
		<div class="analysis-sub-sub-div-1" id="periodText" style=""></div>
	</div>

	<div class="analysis-sub-div-2" id="customerTextDiv" style="height: 75px; width: 100%; top: 130px">
		<div class="analysis-sub-sub-div-2" id="customerText" style=""></div>
	</div>

	<div class="analysis-sub-div-1" id="productTextDiv" style="height: 75px; width: 100%; top: 205px">
		<div class="analysis-sub-sub-div-1" id="productText" style=""></div>
	</div>

	<div class="analysis-sub-div-2" id="radiusTextDiv" style="height: 170px; width: 100%; top: 280px">
		<svg width=100% height=100%>
			<path d={"M 90 40 l0.0001 0"} stroke="black" style={`stroke-width: ${minRadius}; stroke-linecap: round; opacity: 0.3;`}/>
			<path d={"M 90 40 l0.0001 0"} stroke="black" style={`stroke-width: ${4}; stroke-linecap: round;`}/>

			<path d={"M 300 40 l0.0001 0"} stroke="black" style={`stroke-width: ${maxRadius}; stroke-linecap: round; opacity: 0.3;`}/>
			<path d={"M 300 40 l0.0001 0"} stroke="black" style={`stroke-width: ${4}; stroke-linecap: round;`}/>
		</svg>
		<div class="analysis-sub-sub-div-2" id="small-default-radius-text" style=" width: 125px; left: 35px; top: 75px;">corresponds to:</div>
		<div class="analysis-sub-sub-div-2" id="small-radius-text" style="width: 150px; display: grid; place-items: center; left: 23px; top: 105px;">x sales</div>

			
		<div class="analysis-sub-sub-div-2" id="big-default-radius-text" style="width: 125px; left: 235px; top: 75px;">corresponds to:</div>
		<div class="analysis-sub-sub-div-2" id="big-radius-text" style="width: 150px; display: grid; place-items: center; left: 223px; top: 105px;">y sales</div>
	</div>

	<div class="analysis-sub-div-1" id="histogramDiv" style="height: 260px; width: 100%; top: 450px">
		<div class="analysis-sub-sub-div-1" id="histogramDescription" style="display: grid; place-items: center; top: 18px;"> description </div>
		

		<div id="histogramSubDiv" style={`border-radius: 0; border-bottom: 2px solid black; border-left: 2px solid black;  position: absolute; left: 12.5%; top: 22%; width: ${histogramWidth}px; height: ${maxHeightOfBin}px`}>
			<svg style="width: 100%; height: 100%;">
				{#each binValueForHistogram as value, i}
					<!-- svelte-ignore a11y-no-static-element-interactions -->
					<!-- svelte-ignore a11y-mouse-events-have-key-events -->
					<rect 
						width={barWidth}
						height={rescaleHeightOfBin(value)}
						x={i * barWidth}
						y={maxHeightOfBin - rescaleHeightOfBin(value)}
						fill="#0074e1" 
						stroke="#bbdeff"
						on:mouseover={function(event) {selectedHistogramIndex = i; setMousePosition(event);}}
						on:mouseout={function() {selectedHistogramIndex = undefined}}
					/>
				{/each}
			</svg>
			<div style="position: absolute; left: -14px; top: {maxHeightOfBin + 4}px;">
				<strong>0</strong>
			</div>

			<div id="valueOfHistogramDescription" style="display: grid; place-items: center;  font-size: 14px; position: absolute; width: 100%; top: {maxHeightOfBin + 4}px;">
				
			</div>

			<div id="maxValueHistogram" style="position: absolute; left: {histogramWidth-20}px; top: {maxHeightOfBin + 4}px;">
				<strong>0</strong>
			</div>

			<div id="maxYValueHistogram" style="display: grid; place-items: center end; position: absolute; left: -50px; top: -5px; width: 42px">
				<strong>705</strong>
			</div>

			<div style="transform: rotate(-90deg); font-size: 14px; position: absolute; left: -90px; top: 60px; width: 150px">
				Number of cities
			</div>
		</div>
	</div>
</div>

{#if selectedHistogramIndex != undefined}
	<div class="mouse-hover-div" style="left: {mouse_x + 10}px; top: {mouse_y - 10 + scroll_y}px">
		{#if selectedTypeOfDataText == "sales"}
			start range: {binRangesForHistogram[selectedHistogramIndex].toFixed(0).replace(/\B(?=(\d{3})+(?!\d))/g, ",")}<br>
			end range: {binRangesForHistogram[selectedHistogramIndex+1].toFixed(0).replace(/\B(?=(\d{3})+(?!\d))/g, ",")}<br>
		{:else}
			start range: {binRangesForHistogram[selectedHistogramIndex].toFixed(2)}<br>
			end range: {binRangesForHistogram[selectedHistogramIndex+1].toFixed(2)}<br>
		{/if}
		value: {binValueForHistogram[selectedHistogramIndex]}
	</div>
{/if}
