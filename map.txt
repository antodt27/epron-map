// Initialisation de la carte centrée sur Épron
const map = L.map('map').setView([49.222, -0.372], 14);

// Fond de carte OpenStreetMap
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 19,
    attribution: '© OpenStreetMap contributors | Mairie d\'Épron'
}).addTo(map);

// Groupe de clusters pour la gestion des marqueurs
const markersCluster = L.markerClusterGroup();
map.addLayer(markersCluster);

// Dictionnaire de stockage des marqueurs par catégorie
const categoryLayers = {};

// Configuration des catégories (Couleurs + Icônes FontAwesome)
const categoryConfig = {
    'Services publics': { color: '#0056b3', icon: 'fa-landmark' },
    'Éducation':        { color: '#28a745', icon: 'fa-graduation-cap' },
    'Sports & loisirs': { color: '#fd7e14', icon: 'fa-futbol' },
    'Social':           { color: '#e83e8c', icon: 'fa-hands-helping' },
    'Culture':          { color: '#6f42c1', icon: 'fa-book' },
    'Patrimoine':       { color: '#6c757d', icon: 'fa-church' },
    'Cimetière':        { color: '#343a40', icon: 'fa-cross' }
};

// Fonction pour créer une icône personnalisée
function createCustomIcon(category) {
    const config = categoryConfig[category] || { color: '#007bff', icon: 'fa-map-marker-alt' };
    
    return L.divIcon({
        className: 'custom-map-pin',
        html: `<div style="
            background-color: ${config.color};
            width: 32px;
            height: 32px;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            border: 2px solid white;
            box-shadow: 0 2px 5px rgba(0,0,0,0.3);
            color: white;
            font-size: 14px;">
            <i class="fa-solid ${config.icon}"></i>
        </div>`,
        iconSize: [32, 32],
        iconAnchor: [16, 16],
        popupAnchor: [0, -16]
    });
}

// Génération de la Popup Conditionnelle
function buildPopupHTML(properties, lat, lng) {
    const p = properties;
    
    let html = `<div class="popup-header">`;
    if (p.catégorie) {
        html += `<span class="popup-category">${p.catégorie}</span>`;
    }
    html += `<h3>${p.nom || 'Lieu sans nom'}</h3></div>`;

    html += `<div class="popup-body">`;
    if (p.desc) {
        html += `<div class="popup-row"><i class="fa-solid fa-info-circle"></i><span>${p.desc}</span></div>`;
    }
    if (p.adresse) {
        html += `<div class="popup-row"><i class="fa-solid fa-location-dot"></i><span>${p.adresse}</span></div>`;
    }
    if (p.tel) {
        html += `<div class="popup-row"><i class="fa-solid fa-phone"></i><a href="tel:${p.tel.replace(/\s+/g, '')}">${p.tel}</a></div>`;
    }
    if (p.lien) {
        html += `<div class="popup-row"><i class="fa-solid fa-globe"></i><a href="${p.lien}" target="_blank">Site internet / Fiche</a></div>`;
    }
    html += `</div>`;

    const gmapsUrl = `https://www.google.com/maps/dir/?api=1&destination=${lat},${lng}`;
    html += `<div class="popup-footer">
                <a href="${gmapsUrl}" target="_blank"><i class="fa-solid fa-route"></i> Itinéraire vers ce lieu</a>
             </div>`;

    return html;
}

// Création du panneau de filtres dynamique avec "Tout cocher / décocher"
function createFilterControl(categoriesPresentes) {
    const filterControl = L.control({ position: 'topright' });

    filterControl.onAdd = function () {
        const div = L.DomUtil.create('div', 'filter-control');
        L.DomEvent.disableClickPropagation(div);
        L.DomEvent.disableScrollPropagation(div);

        let html = `<h4>Catégories</h4>`;
        
        // Option "Tout cocher / décocher"
        html += `
            <label class="filter-item" style="font-weight: bold; border-bottom: 1px solid #eee; padding-bottom: 6px; margin-bottom: 8px;">
                <input type="checkbox" checked id="toggle-all-checkbox">
                <span>Tout cocher / décocher</span>
            </label>
        `;

        categoriesPresentes.forEach(cat => {
            const config = categoryConfig[cat] || { color: '#007bff' };
            html += `
                <label class="filter-item">
                    <input type="checkbox" checked value="${cat}" class="category-filter-checkbox">
                    <span class="filter-color-dot" style="background-color: ${config.color};"></span>
                    ${cat}
                </label>
            `;
        });

        div.innerHTML = html;
        return div;
    };

    filterControl.addTo(map);

    const toggleAllBtn = document.getElementById('toggle-all-checkbox');
    const categoryCheckboxes = document.querySelectorAll('.category-filter-checkbox');

    // Clic sur "Tout cocher / décocher"
    toggleAllBtn.addEventListener('change', (e) => {
        const isChecked = e.target.checked;

        categoryCheckboxes.forEach(checkbox => {
            checkbox.checked = isChecked;
            const cat = checkbox.value;

            if (categoryLayers[cat]) {
                if (isChecked) {
                    markersCluster.addLayers(categoryLayers[cat]);
                } else {
                    markersCluster.removeLayers(categoryLayers[cat]);
                }
            }
        });
    });

    // Clic sur chaque filtre individuel
    categoryCheckboxes.forEach(checkbox => {
        checkbox.addEventListener('change', (e) => {
            const cat = e.target.value;
            const isChecked = e.target.checked;

            if (categoryLayers[cat]) {
                if (isChecked) {
                    markersCluster.addLayers(categoryLayers[cat]);
                } else {
                    markersCluster.removeLayers(categoryLayers[cat]);
                }
            }

            const allChecked = Array.from(categoryCheckboxes).every(cb => cb.checked);
            toggleAllBtn.checked = allChecked;
        });
    });
}

// Chargement du GeoJSON
fetch('pois_epron_2.geojson')
    .then(response => response.json())
    .then(data => {
        const categoriesPresentes = new Set();
        const allMarkers = [];

        L.geoJSON(data, {
            pointToLayer: function (feature, latlng) {
                const category = feature.properties.catégorie || 'Autre';
                const marker = L.marker(latlng, { icon: createCustomIcon(category) });

                if (!categoryLayers[category]) {
                    categoryLayers[category] = [];
                }
                categoryLayers[category].push(marker);
                categoriesPresentes.add(category);

                return marker;
            },
            onEachFeature: function (feature, layer) {
                if (feature.properties) {
                    const lat = layer.getLatLng().lat;
                    const lng = layer.getLatLng().lng;
                    
                    // Attachement du Popup
                    layer.bindPopup(buildPopupHTML(feature.properties, lat, lng), {
                        className: 'custom-popup'
                    });

                    // Ouverture au survol de la souris
                    layer.on('mouseover', function () {
                        this.openPopup();
                    });
                }
                allMarkers.push(layer);
            }
        });

        markersCluster.addLayers(allMarkers);

        createFilterControl(Array.from(categoriesPresentes).sort());

        const isMobile = window.innerWidth <= 600;
        const bounds = markersCluster.getBounds();
        if (bounds.isValid()) {
            map.fitBounds(bounds, { padding: isMobile ? [15, 15] : [40, 40] });
        }
    })
    .catch(error => console.error('Erreur lors du chargement du GeoJSON :', error));