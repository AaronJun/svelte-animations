<script lang="ts">
	import { onMount } from 'svelte';
	import HeroSection from '$lib/landing/HeroSection.svelte';
	import DataSourcesSection from '$lib/landing/DataSourcesSection.svelte';
	import BentoGridCombo from './bento-grid/BentoGridCombo.svelte';
	import Testimonials from '$lib/landing/Testimonials.svelte';
	import AnalysisCapabilities from '$lib/landing/AnalysisCapabilities.svelte';
	import FeaturesSection from '$lib/landing/FeaturesSection.svelte';
	import SectionTracker from '$lib/landing/SectionTracker.svelte';
	import ContactForm from '$lib/layouts/ContactForm.svelte';

	let activeSection = 'hero';
	let hamburgerMenuIsOpen = false;

	onMount(() => {
		const options = {
			root: null,
			rootMargin: '-50% 0px',
			threshold: 0
		};

		const observer = new IntersectionObserver((entries) => {
			entries.forEach((entry) => {
				if (entry.isIntersecting) {
					activeSection = entry.target.id;
				}
			});
		}, options);

		const sections = ['hero', 'data-sources', 'use-cases', 'analysis','contactform'];
		sections.forEach((sectionId) => {
			const element = document.getElementById(sectionId);
			if (element) observer.observe(element);
		});

		return () => observer.disconnect();
	});

	function scrollToSection(sectionId: string) {
		const element = document.getElementById(sectionId);
		if (element) {
			element.scrollIntoView({ behavior: 'smooth' });
		}
		if (hamburgerMenuIsOpen) {
			hamburgerMenuIsOpen = false;
		}
	}
</script>

<SectionTracker 
	{activeSection}
	{hamburgerMenuIsOpen}
	onSectionClick={scrollToSection}
/>

<div class="pt-2"> <!-- Add padding to account for fixed header -->
	<section id="hero">
		<HeroSection />
	</section>

	<section id="data-sources">
		<DataSourcesSection />
	</section>

<!-- 	
		<BentoGridCombo />
	</section> -->
	<section id="use-cases">
	<FeaturesSection />

	<section id="analysis">
		<AnalysisCapabilities />
	</section>

	<section id="contactform">
	<ContactForm />
	</section>
	
</div>